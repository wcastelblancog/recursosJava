# Guía paso a paso — Week 11: eliminar el N+1 (proyecto solución)

Proyecto: `week-11-performance-orm-solucion` (Java 21, Spring Boot 4.1.1, Hibernate 7, H2 en memoria).
Rutas relativas a la raíz del proyecto. Paquete base: `src/main/java/com/indra/jpa/orders`.

---

## 0. Requisitos y cómo ejecutar

El JDK por defecto de la máquina es 17; este proyecto necesita **JDK 21**. En PowerShell, en cada terminal nueva:

```powershell
cd "C:\Users\wcastelblanco\Documents\BckpPCIndra\Mentoria\2026\ORM HIbernate\week-11-performance-orm-solucion"
$env:JAVA_HOME = "C:\Program Files\Java\jdk-21.0.2"
mvn -v        # debe decir "Java version: 21.0.2"
```

| Acción | Comando |
|--------|---------|
| Compilar + todos los tests | `mvn verify` |
| Solo la clase de conteo | `mvn verify -Dtest=OrderQueryCountTest` |
| Un test concreto | `mvn verify -Dtest=OrderQueryCountTest#listingOrdersExecutesAtMostThreeQueries` |
| Arrancar la app con log SQL | `mvn spring-boot:run "-Dspring-boot.run.profiles=dev"` |
| Limpiar | `mvn clean` |

Para ver el número de queries en consola de los tests, busca la línea `>>> Queries para 50 órdenes: N` en la salida de `mvn verify`.

---

## 1. Entender el punto de partida (el problema)

Modelo: `Order 1─* OrderItem *─1 Product`, todo `LAZY`.
`DataSeeder` crea 100 órdenes × 3 ítems × 1 producto distinto cada uno (en tests: 50 órdenes).

El código base hace:
1. `OrderRepository.findByStatus(status)` → 1 query de órdenes.
2. `OrderView.from(order)` recorre `order.getItems()` → 1 query **por orden**.
3. y lee `item.getProduct().getName()` → 1 query **por ítem**.

Total: `1 + N + 3N` → 50 órdenes = **201**, 100 órdenes = **401**.

---

## 2. Paso 1 — Medir ANTES del fix (diagnóstico)

> En el proyecto solución ya está arreglado. Para reproducir el "antes": en `OrderService.listByStatus(OrderStatus)` cambia temporalmente `findByStatusWithItems` por `findByStatus`, ejecuta el paso 2.3 y vuelve a dejarlo.

### 2.1 Activar log de SQL
Archivo `src/main/resources/application-dev.properties` (ya activo en la solución):

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```

### 2.2 Ver el N+1 en la app
```powershell
mvn spring-boot:run "-Dspring-boot.run.profiles=dev"
```
En otra terminal (o navegador): `curl http://localhost:8081/api/orders`

Qué buscar en la consola: el mismo `select ... from order_items where order_id=?` repetido 100 veces y luego `select ... from products where id=?` repetido 300 veces. Al final de la petición, la estadística de Hibernate indica `401 JDBC statements prepared`.

### 2.3 Medirlo en test
```powershell
mvn verify -Dtest=OrderQueryCountTest#diagnosticPrintsQueryCount
```
Salida con el código base: `>>> Queries para 50 órdenes: 201`. **Anótalo** (va en la descripción del PR).

---

## 3. Paso 2 — Corregir el listado con JOIN FETCH

### 3.1 Repositorio
Archivo `repository/OrderRepository.java`, agrega:

```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
```
Imports: `org.springframework.data.jpa.repository.Query`, `org.springframework.data.repository.query.Param`.

Por qué cada parte:
- `JOIN FETCH o.items` → trae los ítems en la misma query.
- `JOIN FETCH i.product` → trae los productos; sin esto quedarían 3N queries.
- `DISTINCT` → evita órdenes duplicadas (una fila por ítem).
- Es `INNER JOIN`: órdenes sin ítems no aparecen; usa `LEFT JOIN FETCH` si deben aparecer.

### 3.2 Servicio
Archivo `service/OrderService.java`:

```java
@Transactional(readOnly = true)
public List<OrderView> listByStatus(OrderStatus status) {
    return orderRepository.findByStatusWithItems(status).stream().map(OrderView::from).toList();
}
```

### 3.3 Verificar
```powershell
mvn verify -Dtest=OrderQueryCountTest#diagnosticPrintsQueryCount
```
Esperado: `>>> Queries para 50 órdenes: 1`.

### 3.4 No hacer
- **No** cambiar a `FetchType.EAGER` (prohibido: no arregla JPQL y carga siempre).
- **No** usar `@Data` en entidades.

---

## 4. Paso 3 — Alternativa con @EntityGraph (bonus)

En `OrderRepository`:

```java
@EntityGraph(attributePaths = { "items", "items.product" })
List<Order> findWithGraphByStatus(OrderStatus status);
```
Import: `org.springframework.data.jpa.repository.EntityGraph`.

En `OrderService`:

```java
@Transactional(readOnly = true)
public List<OrderView> listByStatusWithGraph(OrderStatus status) {
    return orderRepository.findWithGraphByStatus(status).stream().map(OrderView::from).toList();
}
```

Resultado esperado: 1 query. Usa `LEFT JOIN FETCH` (no oculta órdenes sin ítems).

---

## 5. Paso 4 — Paginación sin N+1 (dos pasos)

### 5.1 Por qué no basta JOIN FETCH
`JOIN FETCH` de una colección + `Pageable` provoca el warning `HHH90003004` y Hibernate **pagina en memoria** (trae todo y recorta en Java). No lo uses así.
Sin ningún fix, una página de 20 cuesta **82 queries**.

### 5.2 Repositorio
```java
@Query(value = "SELECT o.id FROM Order o WHERE o.status = :status ORDER BY o.id",
       countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status")
Page<Long> findIdsByStatus(@Param("status") OrderStatus status, Pageable pageable);

@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.id IN :ids")
List<Order> findWithItemsByIdIn(@Param("ids") List<Long> ids);
```
- Paso A: página de **ids** (LIMIT real en SQL + count).
- Paso B: fetch completo solo de esos ids.
- `ORDER BY o.id` es obligatorio (paginación estable).

### 5.3 Servicio
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
Imports: `java.util.Map`, `java.util.function.Function`, `java.util.stream.Collectors`, `com.indra.jpa.orders.model.Order`.

Por qué el `Map`: la query del paso B no garantiza orden; `ids.map(...)` conserva el orden y los totales de la página.

Queries: ids (1) + count (1) + fetch (1) = **3**.

El `OrderController` (`/api/orders/paged`) no necesita cambios: ya llama a `orderService.listByStatus(status, PageRequest)`.

---

## 6. Paso 5 — Activar y completar los tests

Archivo `src/test/java/com/indra/jpa/orders/OrderQueryCountTest.java`.

1. Elimina los dos `@Disabled` (y su import `org.junit.jupiter.api.Disabled`) de:
   - `listingOrdersExecutesAtMostThreeQueries`
   - `pagedListingExecutesAtMostThreeQueries`
2. Cada uno afirma:
   ```java
   assertThat(measured.queries()).isLessThanOrEqualTo(3);
   ```
3. Tests extra recomendados (ya incluidos en la solución):
   ```java
   @Test
   void entityGraphAlternativeExecutesAtMostThreeQueries() {
       var measured = counter.measure(() -> orderService.listByStatusWithGraph(OrderStatus.CONFIRMED));
       assertThat(measured.queries()).isLessThanOrEqualTo(3);
   }

   @Test
   void pagedListingKeepsOrderTotalAndSize() {
       var page = orderService.listByStatus(OrderStatus.CONFIRMED, PageRequest.of(1, 20));
       assertThat(page.getContent()).hasSize(20);
       assertThat(page.getTotalElements()).isEqualTo(ORDERS);
       assertThat(page.getContent().get(0).customerName()).isEqualTo("Cliente 21");
   }
   ```

Cómo funciona la medición (`QueryCounter`):
- Obtiene `Statistics` del `SessionFactory`.
- `statistics.clear()` antes del bloque, lee `getPrepareStatementCount()` después.
- Requiere `hibernate.generate_statistics=true` (ya en `src/test/resources/application-test.properties`).
- Se crea en `@BeforeEach` **después** del seeding para no contar los inserts.

---

## 7. Validación final

```powershell
mvn clean verify
```

Resultado esperado: `BUILD SUCCESS` y, en los tests:
- listado simple: **1** query
- `@EntityGraph`: **1** query
- paginado (página de 20): **3** queries

Checklist de aceptación:
- [ ] Número de queries **antes** documentado (201 con 50 órdenes / 401 con 100).
- [ ] `JOIN FETCH` implementado en `OrderRepository`.
- [ ] Queries **después** ≤ 3.
- [ ] Tests con aserción explícita del número de queries.
- [ ] Sin `FetchType.EAGER` añadido.
- [ ] Paginado sin warning `HHH90003004`.
- [ ] `mvn verify` en verde.

---

## 8. Bonus — batch fetch size (comparación)

Prueba el impacto sin cambiar código, pasando la propiedad por línea de comandos:

```powershell
mvn verify -Dtest=OrderQueryCountTest "-Dspring.jpa.properties.hibernate.default_batch_fetch_size=25"
```

Medido sobre el mapeo original (sin JOIN FETCH):

| batch size | Listado (50 órdenes) | Paginado (página 20) |
|-----------|----------------------|----------------------|
| sin batch | 201 | 82 |
| 25 | 9 | 6 |
| 100 | 4 | 4 |

Reduce mucho, pero no alcanza ≤ 3 en paginado. Úsalo como red de seguridad global, no como sustituto del fetch explícito.

---

## 9. Problemas frecuentes

| Síntoma | Causa / solución |
|---------|------------------|
| `release version 21 not supported` | Maven usa JDK 17. Ejecuta `$env:JAVA_HOME = "C:\Program Files\Java\jdk-21.0.2"` y revisa `mvn -v`. |
| Test cuenta más queries de las esperadas | Falta `JOIN FETCH i.product`, o el servicio sigue llamando `findByStatus`. |
| Warning `HHH90003004` | Se usó `JOIN FETCH` de colección con `Pageable`. Usa los dos pasos. |
| Órdenes faltantes en la respuesta | `INNER JOIN FETCH` oculta órdenes sin ítems; usa `LEFT JOIN FETCH`. |
| Página en orden raro | Falta `ORDER BY o.id` o no se reordenó con el `Map`. |
| `LazyInitializationException` | Se mapea a DTO fuera del `@Transactional` (con `open-in-view=false`). Mapea dentro del servicio. |
| Puerto 8081 ocupado | Cierra la otra instancia o usa `--server.port=8082`. |