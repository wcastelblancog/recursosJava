# Week 11 – Performance ORM: solución explicada y corregido el N+1

## 1. Objetivo del reto

El problema del reto no es un error de compilación ni un fallo funcional. El objetivo es detectar y corregir el patrón N+1 en una aplicación JPA/Hibernate.

En esta sesión se trabaja con la relación entre `Order`, `OrderItem` y `Product`:

- Una orden tiene muchos ítems
- Cada ítem tiene un producto
- La relación se carga de forma `LAZY` por diseño
- Cuando se recorre esa jerarquía sin plan de carga, Hibernate dispara consultas adicionales por cada entidad hija

La aplicación intenta listar órdenes con sus ítems y el nombre del producto, pero sin optimización termina ejecutando demasiadas consultas SQL.

---

## 2. Qué representa el problema real

El enunciado menciona este escenario:

- 100 órdenes
- cada orden tiene 3 ítems
- cada ítem tiene 1 producto

En el peor caso, con lazy loading, se ejecutan aproximadamente:

- 1 query para obtener las órdenes
- 100 queries para obtener los ítems de cada orden
- 300 queries para obtener los productos de cada ítem

Total estimado: 401 consultas SQL.

Esto es exactamente el patrón N+1:

- N = número de órdenes
- 1 query inicial para las órdenes
- N queries adicionales para items
- N * 3 queries adicionales para productos

El impacto real no es solo del tiempo de ejecución, sino también del consumo de conexiones, memoria y CPU en la base de datos.

---

## 3. La solución en el código fuente

La solución se concentra en estas clases clave:

- `src/main/java/com/indra/jpa/orders/model/Order.java`
- `src/main/java/com/indra/jpa/orders/model/OrderItem.java`
- `src/main/java/com/indra/jpa/orders/model/Product.java`
- `src/main/java/com/indra/jpa/orders/repository/OrderRepository.java`
- `src/main/java/com/indra/jpa/orders/service/OrderService.java`
- `src/test/java/com/indra/jpa/orders/QueryCounter.java`
- `src/test/java/com/indra/jpa/orders/OrderQueryCountTest.java`

---

## 4. Entidades y por qué se genera el N+1

### 4.1 Clase `Order`

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerName;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private LocalDateTime createdAt = LocalDateTime.now();

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
```

Observaciones:

- `Order` tiene una colección `items`
- La relación es `@OneToMany`
- No se usa `FetchType.EAGER`
- Por defecto está en `LAZY`, lo cual es correcto para evitar carga innecesaria

La relación se mantiene así por diseño:

```java
// LAZY a propósito: iterar items dispara 1 query por orden (N+1)
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

### 4.2 Clase `OrderItem`

```java
@Entity
@Table(name = "order_items")
public class OrderItem {

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "product_id")
    private Product product;
```

Aquí ocurre el segundo problema clave:

- `OrderItem.product` es `LAZY`
- Cuando se procesa cada item y se intenta leer el nombre del producto, Hibernate hace otra consulta por cada fila

Eso genera el N+1 anidado:

1. cargar órdenes
2. para cada orden, cargar items
3. para cada item, cargar producto

### 4.3 Clase `Product`

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private BigDecimal price;
```

`Product` es una entidad normal, sin problema propio; el problema aparece cuando se accede a ella desde una relación LAZY que no ha sido pre-cargada.

---

## 5. Qué significa `LAZY` y por qué lo usan

`LAZY` es el comportamiento por defecto en muchas relaciones JPA cuando se trata de asociaciones de colección o entidad.

### Idea general

Hibernate no carga inmediatamente la asociación. La carga se hace solo cuando se accede a ella en memoria.

Ejemplo conceptual:

```java
List<Order> orders = orderRepository.findByStatus(OrderStatus.CONFIRMED);

for (Order order : orders) {
    System.out.println(order.getItems().size());
}
```

En este caso, al entrar a `order.getItems()`, Hibernate puede decidir:

- si la colección no se ha cargado, ejecuta otra query
- y eso ocurre una vez por cada orden

Es decir, el problema no aparece porque la relación está mal declarada, sino porque se accede del modo incorrecto sin cargar los datos en la misma consulta.

### ¿Por qué no se usa `EAGER` como solución?

Porque `EAGER` fuerza carga inmediata para todas las entidades relacionadas y puede empeorar el rendimiento si no se necesita todo en cada consulta.

El reto lo prohíbe explícitamente y la solución correcta es `JOIN FETCH` o `@EntityGraph`.

---

## 6. El dataset de prueba y la observación del problema

La clase `DataSeeder` genera datos deterministas:

```java
@Component
public class DataSeeder {

    public static final int ITEMS_PER_ORDER = 3;

    @Transactional
    public void seed(int orders) {
        for (int i = 1; i <= orders; i++) {
            Order order = new Order("Cliente " + i, OrderStatus.CONFIRMED);
            for (int j = 1; j <= ITEMS_PER_ORDER; j++) {
                Product product = productRepository
                        .save(new Product("Producto " + i + "-" + j, BigDecimal.valueOf(10L * j)));
                order.addItem(new OrderItem(product, j));
            }
            orderRepository.save(order);
        }
    }
}
```

Cada orden crea 3 ítems y cada ítem apunta a un producto distinto.

Eso hace que el problema sea muy visible y reproducible con 50 o 100 órdenes.

---

## 7. Configuración de propiedades para habilitar logs de SQL

En `src/main/resources/application-dev.properties` el proyecto activa la salida SQL y estadísticas de Hibernate:

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.stat=DEBUG
spring.jpa.properties.hibernate.generate_statistics=true
```

Y en `src/test/resources/application-test.properties` también se activa la generación de estadísticas:

```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

### Para qué sirve cada línea

- `spring.jpa.show-sql=true`
  - imprime las queries SQL en consola
- `spring.jpa.properties.hibernate.format_sql=true`
  - formatea el SQL para que sea legible
- `logging.level.org.hibernate.stat=DEBUG`
  - activa logs de estadísticas del ORM
- `spring.jpa.properties.hibernate.generate_statistics=true`
  - habilita contadores para analizar cuántas queries se ejecutaron

Esto es clave para validar antes y después del fix.

---

## 8. Cómo se mide el número de queries

La clase `QueryCounter` es la pieza central para realizar la medición:

```java
public final class QueryCounter {

    private final Statistics statistics;

    public QueryCounter(EntityManagerFactory emf) {
        this.statistics = emf.unwrap(SessionFactory.class).getStatistics();
        this.statistics.setStatisticsEnabled(true);
    }

    public <T> Measured<T> measure(Supplier<T> action) {
        statistics.clear();
        T result = action.get();
        return new Measured<>(result, statistics.getPrepareStatementCount());
    }
```

### Qué hace

- toma el `EntityManagerFactory`
- extrae la estadística del `SessionFactory` de Hibernate
- limpia el contador
- ejecuta la operación a medir
- devuelve cuántas sentencias SQL fueron preparadas

Esto permite afirmar, con evidencia, que el rendimiento mejoró.

---

## 9. La corrección principal: `JOIN FETCH`

La solución se encuentra en `OrderRepository`:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByStatus(OrderStatus status);

    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.status = :status")
    List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
```

### Qué hace esta query

```java
SELECT DISTINCT o
FROM Order o
JOIN FETCH o.items i
JOIN FETCH i.product
WHERE o.status = :status
```

Explicación:

- `FROM Order o` → selecciona las órdenes
- `JOIN FETCH o.items i` → carga la colección de items de la orden en la misma query
- `JOIN FETCH i.product` → carga la relación del producto asociado a cada item
- `DISTINCT` → elimina duplicados generados por el join de una colección
- `WHERE o.status = :status` → filtra por estado

### Por qué corrige el N+1

Antes, Hibernate hacía:

- query de órdenes
- query por cada orden para items
- query por cada item para product

Con JOIN FETCH, Hibernate hace una sola query que recoge todo el árbol necesario para la operación.

Eso elimina la carga perezosa repetida.

---

## 10. Alternativa declarativa: `@EntityGraph`

El repositorio también ofrece una segunda solución alternativa:

```java
@EntityGraph(attributePaths = { "items", "items.product" })
List<Order> findWithGraphByStatus(OrderStatus status);
```

### Qué hace

`@EntityGraph` le dice a JPA/Hibernate:

- por favor, carga estas propiedades asociadas de forma explícita
- no esperes a que el acceso lazy dispare consultas adicionales

Esto puede ser más legible en algunos casos, y funciona muy bien cuando se desea mantener la query más simple.

### Cuándo usarlo

- cuando la relación es conocida y está bien definida
- cuando quieres evitar escribir JPQL más compleja
- cuando el criterio de carga es reutilizable para varios métodos del repositorio

---

## 11. Cómo se consume la solución en la capa de servicio

La clase `OrderService` usa esta estrategia:

```java
@Transactional(readOnly = true)
public List<OrderView> listByStatus(OrderStatus status) {
    return orderRepository.findByStatusWithItems(status).stream()
            .map(OrderView::from)
            .toList();
}
```

Aquí se ve el flujo final:

1. El repositorio trae las órdenes con items y productos ya cargados
2. El servicio transforma la entidad a `OrderView`
3. La API o controlador puede devolver un DTO limpio sin lógica de negocio

---

## 12. Paginación y por qué no se rompe la solución

La paginación se maneja con dos pasos:

```java
@Query(value = "SELECT o.id FROM Order o WHERE o.status = :status ORDER BY o.id",
        countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status")
Page<Long> findIdsByStatus(@Param("status") OrderStatus status, Pageable pageable);

@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items i JOIN FETCH i.product WHERE o.id IN :ids")
List<Order> findWithItemsByIdIn(@Param("ids") List<Long> ids);
```

### Por qué es importante

Si se hace `JOIN FETCH` directo sobre una consulta paginada sin pensar bien el diseño, puede aparecer:

- duplicación de filas
- problemas con `LIMIT/OFFSET`
- resultados inconsistentes con la página

Por eso la solución del reto separa dos pasos:

1. consulta paginada por IDs
2. fetch de los datos relacionados solo para esos IDs

Luego en el servicio:

```java
Page<Long> ids = orderRepository.findIdsByStatus(status, pageable);
Map<Long, Order> byId = orderRepository.findWithItemsByIdIn(ids.getContent()).stream()
        .collect(Collectors.toMap(Order::getId, Function.identity()));
```

Así se garantiza que:

- la página mantiene el orden correcto
- la carga de la colección y producto se hace de forma eficiente
- no se rompe el patrón de paginación

---

## 13. Tests que validan la corrección

El test principal se encuentra en `OrderQueryCountTest`:

```java
@Test
void listingOrdersExecutesAtMostThreeQueries() {
    var measured = counter.measure(() -> orderService.listByStatus(OrderStatus.CONFIRMED));
    assertThat(measured.queries()).isLessThanOrEqualTo(3);
}
```

También hay validación paginada:

```java
@Test
void pagedListingExecutesAtMostThreeQueries() {
    var measured = counter.measure(
            () -> orderService.listByStatus(OrderStatus.CONFIRMED, PageRequest.of(0, 20)));

    assertThat(measured.queries()).isLessThanOrEqualTo(3);
}
```

### Qué demuestra

No solo “funciona”, sino que el comportamiento de acceso a datos queda medido y validado con una aserción explícita.

En un reto de performance, esa es la diferencia entre una solución teoría y una solución verificable.

---

## 14. Cómo quedó corregido el N+1 en la práctica

La solución está enfocada en este punto:

- antes: por cada orden se cargaba la colección y la relación del producto con consultas adicionales
- después: la query de carga principal trae toda la información necesaria en una sola operación

El resultado esperado en pruebas es:

- para 50 órdenes
- máximo 3 consultas SQL
- no más 201 como antes

Eso es lo que se valida en las aserciones del test.

---

## 15. Resumen del flujo completo

### Antes del fix

1. `findByStatus()` devuelve las órdenes
2. el servicio itera por cada orden
3. `items` se carga por cada orden
4. cada `product` se carga por cada item
5. Hibernate hace muchas queries ocultas

### Después del fix

1. `findByStatusWithItems()` ejecuta una query con `JOIN FETCH`
2. Hibernate trae orden + items + productos en la misma consulta
3. El servicio transforma a DTOs sin más accessos lazy problemáticos
4. Las pruebas validan que el número de consultas no excede el umbral

---

## 16. Conclusión

Esta semana resuelve un problema central del desarrollo con JPA/Hibernate: la diferencia entre una carga perezosa “normal” y una carga perezosa “descontrolada”.

La solución correcta no es evitar `LAZY`, ni usar `EAGER` a la fuerza, sino definir una estrategia de fetch consciente para cada caso:

- `JOIN FETCH` para consultas de negocio que requieren datos completos
- `@EntityGraph` como alternativa declarativa
- paginación con carga de IDs + fetch por lote de IDs
- métricas de Hibernate para validar el comportamiento real

En otras palabras, el reto enseña a pensar como un desarrollador que no solo hace funcionar la app, sino que también la hace escalar y responder adecuadamente en producción.

---

## 17. Preguntas frecuentes y respuestas

### P1. ¿Qué es un N+1?

Es un patrón donde una consulta principal dispara más consultas adicionales por cada fila del resultado principal.

En este caso:

- 1 query para órdenes
- +50 para items
- +150 para productos

Eso es N+1 en acción.

### P2. ¿Qué es `LAZY`?

Es la carga diferida de relaciones. Hibernate no carga la asociación al instante; la carga se hace solo cuando se accede a ella.

Es útil, pero si se usa sin control puede producir N+1.

### P3. ¿Por qué no usamos `EAGER`?

Porque `EAGER` carga siempre la relación, aunque no se necesite. Eso puede empeorar el rendimiento en consultas de listado y traer más datos de la base de datos de lo necesario.

### P4. ¿Qué hace `JOIN FETCH`?

Carga la relación asociada dentro de la misma consulta SQL. Es una forma de resolver N+1 sin hacer `EAGER` global.

### P5. ¿Para qué sirve `@EntityGraph`?

Es una alternativa declarativa para definir qué asociaciones deben cargarse con una entidad. Es útil cuando se quiere un plan de fetch claro sin escribir JPQL largo.

### P6. ¿Por qué el test mide queries y no solo tiempo?

Porque el tiempo depende del entorno, la JVM, la base de datos y la carga del contexto. El número de queries es una métrica más estable y representativa del problema de N+1.

### P7. ¿La paginación rompe el `JOIN FETCH`?

Puede romper si se usa sin estrategia. Por eso esta solución trabaja primero con IDs paginados y luego hace fetch solo para esa página.

### P8. ¿Cómo sé que la solución quedó bien?

Cuando el test valida que la cantidad de queries queda dentro del límite esperado y la aplicación continúa funcionando con paginación y datos correctos.

---

## 18. Checklist de validación

- [x] SQL log habilitado
- [x] N+1 identificado en el modelo
- [x] `JOIN FETCH` implementado
- [x] `@EntityGraph` como alternativa
- [x] paginación preservada
- [x] tests de cuenta de queries
- [x] sin `FetchType.EAGER` como solución del problema

Si quieres, en el siguiente paso puedo ayudarte a convertir este documento en una versión aún más formal para PR o para presentación en sesión, con formato tipo “explicación técnica para reviewer” y con ejemplos de comentarios en GitHub.
