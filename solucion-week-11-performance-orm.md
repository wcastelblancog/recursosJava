# Solución de referencia — Week 11: Performance ORM (N+1)

> **Solo para formadores. No publicar en `week-11-start`.**
> Todo lo descrito aquí se **ejecutó y verificó** en una copia del proyecto (JDK 21, Spring Boot 4.1.1, Hibernate 7.4, H2).
> Los números de queries son medidos con `Statistics.getPrepareStatementCount()`, dataset de 50 órdenes × 3 ítems × 1 producto distinto por ítem.

---

## 1. Resumen de resultados medidos

| Escenario | Queries |
|-----------|---------|
| Listado base (N+1), 50 órdenes | **201** (1 + 50 + 150) |
| Listado base, 100 órdenes (dataset de la app) | **401** (1 + 100 + 300) |
| Listado con `JOIN FETCH` | **1** |
| Listado con `@EntityGraph` | **1** |
| Paginado base (`Page<Order>`, página de 20) | **82** |
| Paginado con solución de dos pasos (ids + fetch) | **3** |
| Listado base con `default_batch_fetch_size=25` | 9 |
| Listado base con `default_batch_fetch_size=100` | 4 |
| Paginado base con `default_batch_fetch_size=25` | 6 |
| Paginado base con `default_batch_fetch_size=100` | 4 |

Conclusión: `JOIN FETCH` / `@EntityGraph` resuelven el listado completo; para paginado la solución robusta es **dos pasos**; el batch size reduce mucho pero no llega a ≤ 3 en paginado (el `count` suma una query).

---

## 2. Por qué ocurre el N+1 (explicación detallada)

El modelo es `Order 1──* OrderItem *──1 Product`, todo `LAZY` (lo correcto por defecto):

```java
@OneToMany(mappedBy = "order", ...)            private List<OrderItem> items;   // LAZY
@ManyToOne(fetch = FetchType.LAZY ...)         private Product product;        // LAZY (en OrderItem)
```

`orderRepository.findByStatus(status)` ejecuta **una** query: `select ... from orders where status=?`.
Las entidades `Order` devueltas tienen `items` como un proxy (`PersistentBag`) sin inicializar.
Después, `OrderView.from(order)` hace:

```java
order.getItems().stream().map(item -> item.getProduct().getName() ...)
```

- `order.getItems()` + recorrerlo → Hibernate lanza `select ... from order_items where order_id=?` **por cada orden** → **N** queries.
- `item.getProduct().getName()` → el `Product` es un proxy; leer `getName()` lanza `select ... from products where id=?` **por cada ítem** → **N × 3** queries.

Total: `1 + N + 3N`. Con 50 órdenes = 201; con 100 = 401.

Por qué no se ve a simple vista: el endpoint devuelve el JSON correcto; el único síntoma es latencia y carga en BD, que crece **linealmente con los datos** (funciona en dev con 5 filas, falla en prod con 10.000).

Por qué `open-in-view=false` y mapear dentro del `@Transactional`: con OSIV activo, la carga LAZY también "funcionaría" desde el controller y el problema quedaría más oculto. Aquí la carga ocurre dentro del servicio, donde se debe corregir.

Por qué las entidades usan `@Getter/@Setter` y no `@Data`: `@Data` genera `toString/equals/hashCode` que recorren las relaciones y disparan cargas LAZY no intencionales (o recursión infinita en bidireccionales).

### Por qué **no** usar `FetchType.EAGER`
- No cambia las queries JPQL/derivadas: Hibernate igualmente ejecuta N queries adicionales para inicializar el EAGER tras la consulta (solo `find by id` usa join).
- Carga siempre la relación, incluso en casos de uso que no la necesitan.
- Es una decisión global del mapeo; un fetch debe decidirse **por caso de uso**.

---

## 3. Paso 1 — Diagnóstico

Archivo `src/main/resources/application-dev.properties` (descomentar):

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```

Ejecutar: `mvn spring-boot:run -Dspring-boot.run.profiles=dev` y `GET http://localhost:8081/api/orders`.

Qué se ve: el mismo `select ... from order_items where order_id=?` repetido, y luego `select ... from products where id=?` repetido. Con `generate_statistics` el log final dice, por ejemplo, `401 JDBC statements prepared`. Ese número es el que se documenta **antes** en el PR.

El test `diagnosticPrintsQueryCount` imprime `>>> Queries para 50 órdenes: 201` usando `QueryCounter`.

---

## 4. Paso 2 — Fix del listado con JOIN FETCH

`OrderRepository`:

```java
@Query("SELECT DISTINCT o FROM Order o " +
       "JOIN FETCH o.items i JOIN FETCH i.product " +
       "WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
```

`OrderService.listByStatus`:

```java
return orderRepository.findByStatusWithItems(status).stream().map(OrderView::from).toList();
```

Explicación:
- `JOIN FETCH o.items` hace un `INNER JOIN` y **hidrata la colección** en la misma query.
- `JOIN FETCH i.product` encadena el fetch hasta el producto: sin él, los productos seguirían siendo N×3 queries.
- `DISTINCT` evita que cada orden aparezca repetida (una fila por ítem). En Hibernate 6+ la deduplicación de la entidad raíz es automática; mantenerlo es inocuo y explícito para quien viene de versiones anteriores.
- Es `INNER JOIN`: una orden **sin ítems no aparecería**. Si el negocio lo requiere, usar `LEFT JOIN FETCH o.items`.
- Resultado medido: **1 query**.

## 5. Variante con @EntityGraph (bonus)

```java
@EntityGraph(attributePaths = { "items", "items.product" })
List<Order> findWithGraphByStatus(OrderStatus status);
```

- Declarativo: reutiliza queries derivadas sin escribir JPQL.
- Usa `LEFT JOIN FETCH` (no excluye órdenes sin ítems).
- Mismo resultado medido: **1 query**.
- Cuándo usar cuál: `@EntityGraph` para queries derivadas simples; `JOIN FETCH` cuando la query ya es JPQL custom o necesitas control del tipo de join.

---

## 6. Paso 3 — El listado paginado (la parte difícil)

Sin tocar nada, `Page<Order> findByStatus(status, pageable)` + el mapeo a DTO da **82 queries** para una página de 20 (1 órdenes + 1 count + 20 ítems + 60 productos).

### Trampa: `JOIN FETCH` + colección + `Pageable`
Si se escribe:

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
Page<Order> bad(..., Pageable p);
```

Hibernate **no puede aplicar `LIMIT`** sobre un join con colección (las filas se multiplican por ítem). Emite el warning `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory` y **trae TODAS las filas a memoria** y pagina allí. Pasa los tests con pocos datos, pero revienta en producción. Un revisor debe rechazarlo.

### Solución recomendada: dos pasos
1. Paginar **solo ids** (query ligera, `LIMIT` real en SQL, más el count).
2. Cargar con `JOIN FETCH` las órdenes de esos ids.

`OrderRepository`:

```java
@Query(value = "SELECT o.id FROM Order o WHERE o.status = :status ORDER BY o.id",
       countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status")
Page<Long> findIdsByStatus(@Param("status") OrderStatus status, Pageable pageable);

@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.id IN :ids")
List<Order> findWithItemsByIdIn(@Param("ids") List<Long> ids);
```

`OrderService`:

```java
@Transactional(readOnly = true)
public Page<OrderView> listByStatus(OrderStatus status, Pageable pageable) {
    Page<Long> ids = orderRepository.findIdsByStatus(status, pageable);
    if (ids.isEmpty()) {
        return ids.map(id -> null);
    }
    Map<Long, Order> byId = orderRepository.findWithItemsByIdIn(ids.getContent()).stream()
            .collect(Collectors.toMap(Order::getId, Function.identity()));
    return ids.map(id -> OrderView.from(byId.get(id)));
}
```

Detalles importantes:
- `ORDER BY o.id` es obligatorio: sin orden determinista la paginación es inestable.
- La query del paso 2 **no garantiza orden**, por eso se re-ordena con el `Map` siguiendo el orden de `ids` (`ids.map(...)` conserva el orden de la página).
- `Page.map` conserva `totalElements`, número y tamaño de página (verificado: página 1 → 20 elementos, total 50, empieza en `Cliente 21`).
- Queries: ids (1) + count (1) + fetch (1) = **3** → cumple "≤ 3".

> Nota: `Slice`/`List<Long>` en lugar de `Page` evitaría el count (2 queries) si el cliente no necesita el total.

---

## 7. Bonus — `hibernate.default_batch_fetch_size`

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

En vez de 1 query por entidad, Hibernate inicializa proxies **en lotes** con `where id in (?, ?, ... )`. No modifica los métodos del repositorio y **no rompe `Pageable`**.

Medido (sin `JOIN FETCH`, mapeo original):

| batch size | Listado (50 órdenes) | Paginado (página 20) |
|-----------|----------------------|----------------------|
| sin batch | 201 | 82 |
| 25 | 9 | 6 |
| 100 | 4 | 4 |

Lectura: baja de cientos a un puñado, pero **no llega a ≤ 3** en el paginado (órdenes + count + ítems + productos = 4 como mínimo). Útil como red de seguridad global; para el umbral del reto se necesita el fix por caso de uso. Se pueden combinar: fetch explícito en los casos críticos y batch size como defensa general.

---

## 8. Tests de la solución

Se quita `@Disabled` de los dos tests del reto en `OrderQueryCountTest`:

```java
@Test
void listingOrdersExecutesAtMostThreeQueries() {
    var measured = counter.measure(() -> orderService.listByStatus(OrderStatus.CONFIRMED));
    assertThat(measured.queries()).isLessThanOrEqualTo(3);   // real: 1
}

@Test
void pagedListingExecutesAtMostThreeQueries() {
    var measured = counter.measure(() -> orderService.listByStatus(OrderStatus.CONFIRMED, PageRequest.of(0, 20)));
    assertThat(measured.queries()).isLessThanOrEqualTo(3);   // real: 3
}
```

Y se mantienen los tests de contenido (`listsOrdersWithItemsAndProductNames`) para probar que el fix no cambió el resultado funcional. Un test adicional recomendado: la página 1 devuelve 20 elementos, `totalElements = 50` y respeta el orden.

`mvn verify` resultado: **verde**.

Cómo funciona `QueryCounter`: obtiene `Statistics` del `SessionFactory`, hace `clear()` antes del bloque medido y lee `getPrepareStatementCount()` después. Requiere `hibernate.generate_statistics=true` (ya en `application-test.properties`). Se crea **después** del seeding en el `@BeforeEach` para no contar los inserts.

---

## 9. Idea para nivel experto: detección de N+1 en CI

- **datasource-proxy** o **p6spy**: envuelven el `DataSource` y cuentan/loguean sentencias por petición; un test o filtro falla si supera un umbral.
- **hypersistence-utils** `SQLStatementCountValidator`: aserciones declarativas `assertSelectCount(1)`.
- Integrar en el pipeline de Azure DevOps de `mvn verify` para que un N+1 nuevo rompa el build.

---

## 10. Cómo se evalúa cada nivel (resumen)

| Nivel | Qué esperar |
|-------|-------------|
| Junior | `JOIN FETCH` en el listado simple, logs activados, conteo antes/después en el PR. |
| Semi-senior | Test con aserción del conteo; entiende por qué no EAGER. |
| Senior | Resuelve paginación sin paginar en memoria (dos pasos); compara `@EntityGraph` y batch size. |
| Experto | Propone detección automática de N+1 en CI (datasource-proxy/p6spy). |

## 11. Errores comunes a vigilar en los PR
- Cambiar a `FetchType.EAGER` (prohibido).
- `JOIN FETCH` de colección con `Pageable` (warning `HHH90003004`, paginación en memoria).
- Olvidar `JOIN FETCH i.product` → quedan N×3 queries.
- `INNER JOIN FETCH` que oculta órdenes sin ítems.
- Test que solo comprueba "no lanza excepción" sin aserción del número de queries.
- Perder el orden de la página en el paso 2 de la solución paginada.