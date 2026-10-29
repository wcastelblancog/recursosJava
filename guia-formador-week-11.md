# Guía del formador — Week 11: Performance ORM (N+1)

> Documento para el formador. **No publicar en la rama `week-11-start`** (incluye la solución).

## 1. Qué contiene el proyecto base

Ubicación: `sessions/week-11-performance-orm/` (Java 21, Spring Boot 4.1.1, Hibernate 7, H2 en memoria, Lombok).
Ejecutar con JDK 21: `$env:JAVA_HOME="C:\Program Files\Java\jdk-21.0.2"; mvn verify`.

### Dominio (`com.indra.jpa.orders`)
| Clase | Rol |
|-------|-----|
| `model/Order` | Orden con `customerName`, `status`, `items` (`@OneToMany` LAZY). |
| `model/OrderItem` | Ítem con `order` y `product` (`@ManyToOne` LAZY) y `quantity`. |
| `model/Product` | Producto con `name` y `price`. |
| `model/OrderStatus` | `PENDING`, `CONFIRMED`, `CANCELLED`. |
| `repository/OrderRepository` | `findByStatus` (lista) y `findByStatus(..., Pageable)`. **Aquí vive el N+1.** |
| `repository/ProductRepository` | CRUD de productos (usado por el seeder). |
| `service/OrderService` | `@Transactional(readOnly = true)`; mapea a DTO dentro de la transacción. |
| `service/OrderView` | Record DTO (id, cliente, estado, ítems con nombre de producto, cantidad y precio). |
| `web/OrderController` | `GET /api/orders` y `GET /api/orders/paged?page=&size=`. Sin lógica de negocio. |
| `config/DataSeeder` + `SeedRunner` | Carga 100 órdenes × 3 ítems × 1 producto distinto cada uno. Se ejecuta al arrancar salvo perfil `test`. |

### Decisiones de diseño (para explicarlas)
- **Todo es LAZY a propósito**: es el mapeo correcto; el problema aparece al recorrer las relaciones.
- **`@Getter/@Setter` en vez de `@Data`** en entidades: `toString/equals/hashCode` recorrerían relaciones LAZY y ocultarían o provocarían cargas.
- **`spring.jpa.open-in-view=false`**: evita que una sesión abierta en la vista esconda cargas LAZY.
- **Dataset determinista**: 1 + N órdenes + 3N productos. Con 100 órdenes = **401 queries**; con 50 = **201** (medido).
- **`application-dev.properties`** trae el log de SQL **comentado**: es el paso 1 del reto.

### Tests (`src/test`)
- `QueryCounter`: usa `Statistics.getPrepareStatementCount()` de Hibernate (`generate_statistics=true` en `application-test.properties`).
- `OrderQueryCountTest`:
  - `listsOrdersWithItemsAndProductNames`: valida el contenido (debe seguir en verde tras el fix).
  - `diagnosticPrintsQueryCount`: imprime `>>> Queries para 50 órdenes: 201` (para la demo).
  - `listingOrdersExecutesAtMostThreeQueries` y `pagedListingExecutesAtMostThreeQueries`: **`@Disabled`**; los estudiantes los activan.

## 2. Solución de referencia (propuesta)

> El lado del listado simple está alineado con el enunciado. La parte paginada y las alternativas son propuestas
> que **no se ejecutaron** en el repo todavía: pruébalas antes de la sesión.

### 2.1 Diagnóstico
En `application-dev.properties` descomentar:
```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```
Arrancar con `mvn spring-boot:run -Dspring-boot.run.profiles=dev`, llamar `/api/orders` y contar sentencias (401). Documentar el número en el PR.

### 2.2 Fix con JOIN FETCH (listado)
```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
```
Y en `OrderService.listByStatus` usar `findByStatusWithItems`. Resultado esperado: **1 query**.

### 2.3 Alternativa con @EntityGraph (bonus)
```java
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByStatus(OrderStatus status);
```

### 2.4 Paginación (punto delicado)
`JOIN FETCH` de una colección + `Pageable` hace que Hibernate pagine **en memoria** (warning `HHH90003004`): funciona pero carga todo. Opciones:
1. **Dos pasos (recomendada)**: página de ids (con count) y luego fetch de esas órdenes:
   ```java
   @Query("SELECT o.id FROM Order o WHERE o.status = :status")
   Page<Long> findIdsByStatus(@Param("status") OrderStatus status, Pageable pageable);

   @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.id IN :ids")
   List<Order> findWithItemsByIdIn(@Param("ids") List<Long> ids);
   ```
   Total esperado: 3 queries (ids, count, fetch). Ojo: conservar el orden de los ids al armar la página.
2. **`hibernate.default_batch_fetch_size=25`** (bonus): no rompe `Pageable`, pero la carga son varias queries por lote; con la página de 20 y count probablemente supere 3, así que sirve para comparar impacto, no para el umbral de 3.

### 2.5 Tests
Quitar `@Disabled` de los dos tests de conteo; ambos deben afirmar `queries <= 3`. El test de contenido sigue verde.

## 3. Discurso para los estudiantes (≈ 20 min de demo + Q&A)

**Apertura (2')**
"Hoy atacamos la brecha más grande de Hibernate/ORM en el QC (64 %): el problema N+1. Es el problema de performance más común con ORM, es fácil de introducir y casi invisible si no mides. Un endpoint que 'funciona' puede lanzar 401 queries y tardar 3 segundos."

**Demo 1 — Verlo (6')**
1. "Este es el proyecto base: órdenes, ítems y productos. Todo LAZY, que es lo correcto."
2. Arrancar en perfil `dev` con el log de SQL activado y llamar `/api/orders`.
3. "Miren la consola: una query para las órdenes y luego el mismo `select` de ítems una y otra vez, y después uno por cada producto. 1 + 100 + 300 = 401."
4. Mostrar el test `diagnosticPrintsQueryCount`: "No basta con ver logs; en el PR queremos un número y una aserción."

**Demo 2 — Por qué pasa (4')**
"LAZY crea proxies. Al recorrer `order.getItems()` y `item.getProduct()` Hibernate consulta bajo demanda. El mapeo a DTO en el servicio es donde se materializa. Y ojo: **cambiar a EAGER no es solución**, solo mueve el problema y lo empeora en otros casos. Está prohibido en el reto."

**Demo 3 — Las herramientas (5')**
"Tienen tres formas: `JOIN FETCH` en JPQL, `@EntityGraph`, y `default_batch_fetch_size`. Con colecciones y paginación hay una trampa: Hibernate pagina en memoria y avisa con `HHH90003004`. Piensen cómo resolver el listado paginado."
(No mostrar la solución completa; solo el patrón de `JOIN FETCH` y la advertencia.)

**El reto (3')**
"Su tarea, entre semana, en una rama `week-11/<alias>` y con PR:
1. Activen el log de SQL en `application-dev.properties` y documenten cuántas queries hay **antes**.
2. Corrijan el N+1 en `OrderRepository` con `JOIN FETCH` (sin EAGER).
3. Quiten el `@Disabled` de los tests de conteo: máximo **3 queries**, tanto en el listado como en el paginado.
4. `mvn verify` en verde, con Java 21.
5. Un compañero revisa su PR con el checklist de `criterios-evaluacion.md`."

**Cómo se evalúa el nivel (1')**
"Todos hacen el mismo reto. Un junior entrega el `JOIN FETCH` funcionando. Un senior demuestra **cómo**: test con métricas, paginación bien resuelta, `@EntityGraph` o batch size comparados. Un experto propone detectar N+1 en CI (datasource-proxy o p6spy)."

**Cierre y Q&A**
"El PR se entrega antes del próximo jueves; ese día revisamos `week-11-solution`. Quien no asistió puede resolver con `week-11-start`. Preguntas."

## 4. Preguntas frecuentes esperadas
- *¿Por qué no EAGER?* Carga siempre, aunque no lo uses, y no evita N+1 con consultas JPQL.
- *¿Por qué `DISTINCT`?* El join con colecciones duplica la entidad raíz por cada ítem.
- *¿Por qué falla la paginación con JOIN FETCH?* Hibernate no puede aplicar `LIMIT` sobre filas duplicadas; pagina en memoria.
- *El test dice 0 queries / más de las esperadas.* Revisar que `generate_statistics=true` y que `QueryCounter` se crea después del seeding.

## 5. Checklist del formador antes de la sesión
- [ ] `mvn verify` en verde con JDK 21.
- [ ] Probar la solución paginada de 2.4 y ajustar este documento.
- [ ] Publicar `week-11-start` sin esta guía ni la solución.