# PHM Semana 4 — Kotlin + Spring Boot: Persistencia ORM/JPA, Tests de Integración y Serialización

> Notas de clase expandidas con contexto técnico, funcionamiento interno y recursos útiles.

---

## Tabla de contenidos

1. [Inyección de dependencias — `@Autowired`](#1-inyección-de-dependencias--autowired)
2. [Transaccionalidad — `@Transactional`](#2-transaccionalidad--transactional)
3. [Configuración — `application.yml`](#3-configuración--applicationyml)
4. [Objetos de dominio — JPA/Hibernate](#4-objetos-de-dominio--jpahíbernate)
5. [Estrategias de serialización y deserialización](#5-estrategias-de-serialización-y-deserialización)
6. [Tests de integración con MockMVC](#6-tests-de-integración-con-mockmvc)
7. [Links útiles](#7-links-útiles)

---

## 1. Inyección de dependencias — `@Autowired`

```kotlin
@RestController
class BookController(@Autowired private val bookService: BookService)
```

### ¿Qué hace internamente?

Spring mantiene un **contenedor IoC** (Inversion of Control) que gestiona el ciclo de vida de los beans. Cuando una clase está anotada con `@Component`, `@Service`, `@Repository` o `@RestController`, Spring la registra como bean. `@Autowired` le indica al contenedor que inyecte una instancia ya existente en ese punto.

Por defecto, todos los beans son **singletons** dentro del contexto de la aplicación: Spring crea una única instancia y la reutiliza en cada inyección. Esto implica que el estado mutable en un bean inyectado es compartido entre todas las peticiones — hay que tener cuidado con variables de instancia no thread-safe.

### Jerarquía típica

```
Controller  →  Service  →  Repository  →  Base de datos
                    ↘  OtherService
```

### Consejo práctico

En Kotlin, la forma idiomática es usar **inyección por constructor** en lugar de `@Autowired` en el campo, lo que además facilita el testing:

```kotlin
// ✅ Preferido en Kotlin
@Service
class BookService(
    private val bookRepository: BookRepository,
    private val authorService: AuthorService
)

// ⚠️ Funciona pero menos idiomático
@Service
class BookService {
    @Autowired
    private lateinit var bookRepository: BookRepository
}
```

> 📖 [Spring IoC Container — Documentación oficial](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans)

---

## 2. Transaccionalidad — `@Transactional`

### Principio base

Una transacción es una unidad de trabajo **atómica**: o se completa entera, o se deshace por completo (rollback). Esto garantiza consistencia en la base de datos ante fallos parciales.

```kotlin
@Transactional
fun transferFunds(fromId: Long, toId: Long, amount: Double) {
    val from = accountRepository.findById(fromId)
    val to   = accountRepository.findById(toId)
    from.balance -= amount  // si esto falla...
    to.balance   += amount  // ...esto se revierte también
    accountRepository.save(from)
    accountRepository.save(to)
}
```

### Variantes importantes

| Anotación | Efecto |
|---|---|
| `@Transactional` | Abre una transacción; hace rollback en `RuntimeException` |
| `@Transactional(readOnly = true)` | Deshabilita el *dirty checking* de Hibernate → mejor performance en consultas |
| `@Transactional(rollbackFor = [Exception::class])` | Incluye excepciones chequeadas en el rollback |
| `@Transactional(propagation = Propagation.REQUIRES_NEW)` | Abre una nueva transacción independiente |

### Excepciones chequeadas vs. no chequeadas

Esta distinción es **crítica** en Spring:

- **`RuntimeException` y subclases** (no chequeadas) → **SÍ** disparan rollback por defecto.
- **`Exception` y subclases chequeadas** (ej. `IOException`, `SQLException`) → **NO** disparan rollback por defecto. Para incluirlas: `@Transactional(rollbackFor = [Exception::class])`.

```
Throwable
├── Error                      (no chequeada — JVM errors)
└── Exception
    ├── RuntimeException       ← Spring hace rollback aquí por defecto
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   └── ...
    └── IOException            ← NO hace rollback salvo configuración explícita
        └── SQLException
```

### ¿Cómo funciona internamente?

Spring envuelve el bean en un **proxy dinámico**. Al llamar a un método `@Transactional`, el proxy intercepta la llamada, abre una conexión con la DB, ejecuta el método real, y según el resultado hace `commit` o `rollback`. Por eso las llamadas `@Transactional` **dentro de la misma clase** no pasan por el proxy y no respetan la anotación — hay que llamarlas desde otra clase.

> 📖 [Spring @Transactional — Baeldung (muy completo)](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring)

---

## 3. Configuración — `application.yml`

Ubicación: `src/main/resources/application.yml`

```yaml
spring:
  jpa:
    open-in-view: false          # ← importante
    hibernate:
      ddl-auto: validate         # en producción nunca usar create/create-drop
    show-sql: true               # útil en desarrollo, apagar en producción
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
```

### `open-in-view: false` — ¿por qué importa?

Por defecto Spring Boot tiene `open-in-view=true`, lo que mantiene la sesión de Hibernate **abierta durante todo el ciclo HTTP** (incluso mientras se renderiza la respuesta). Esto permite lazy loading "accidental" fuera de la capa de servicio, generando:

- **N+1 queries** silenciosas que destruyen la performance.
- Acoplamiento entre la capa web y la de persistencia.
- Errores difíciles de detectar en producción.

Con `open-in-view: false`, si intentás acceder a una relación lazy fuera de una transacción activa, obtenés una `LazyInitializationException` — que es **ruido útil** que te fuerza a diseñar mejor tus queries.

### `ddl-auto` en distintos entornos

| Valor | Uso recomendado |
|---|---|
| `create-drop` | Tests (crea y destruye el esquema) |
| `update` | Desarrollo rápido (riesgoso en producción) |
| `validate` | Producción (valida contra el esquema existente) |
| `none` | Producción con migraciones manuales (Flyway/Liquibase) |

> 📖 [Spring Boot Application Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html)

---

## 4. Objetos de dominio — JPA/Hibernate

```kotlin
@Entity
@Table(name = "books")
data class Book(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, length = 255)
    val title: String,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    val author: Author,

    @Enumerated(EnumType.STRING)
    val genre: Genre
)
```

### Anotaciones clave

| Anotación | Descripción |
|---|---|
| `@Entity` | Marca la clase como tabla en la DB |
| `@Table(name = "...")` | Nombre explícito de la tabla (opcional) |
| `@Id` | Clave primaria |
| `@GeneratedValue` | Estrategia de generación del ID (`IDENTITY`, `SEQUENCE`, `AUTO`) |
| `@Column` | Customiza columna: nullable, length, unique, name |
| `@ManyToOne` / `@OneToMany` | Relaciones entre entidades |
| `@JoinColumn` | Columna FK en la tabla |
| `@Enumerated(EnumType.STRING)` | Persiste el enum como string (más legible que ordinal) |

### `FetchType.LAZY` vs `EAGER`

- **LAZY** (recomendado): la relación se carga **solo cuando se accede** al campo. Más eficiente, requiere sesión activa.
- **EAGER**: la relación se carga **siempre**, incluso si no se necesita. Puede generar queries innecesarias.

### Consejo con `data class` en Kotlin

Las `data class` de Kotlin generan `equals`, `hashCode` y `toString` basados en todos los campos. Con entidades JPA esto puede ser problemático (especialmente con relaciones lazy). Opciones:

- Usar `class` regular con `@EqualsAndHashCode` manual.
- Usar el plugin `kotlin-jpa` para generar constructores sin argumentos automáticamente.
- Evitar incluir relaciones en `toString` para prevenir lazy loading accidental.

> 📖 [JPA Annotations Reference](https://jakarta.ee/specifications/persistence/3.1/apidocs/)
> 📖 [Kotlin + Spring Data JPA — Baeldung](https://www.baeldung.com/kotlin/spring-data-jpa)

---

## 5. Estrategias de serialización y deserialización

El problema: las entidades JPA no siempre deben exponerse tal cual al cliente. Hay que controlar qué campos viajan en el JSON.

### A) DTO (Data Transfer Object) — ✅ Recomendado

Clases separadas que representan exactamente lo que entra y sale de la API.

```kotlin
// Lo que recibo del cliente
data class CreateBookRequest(
    val title: String,
    val authorId: Long,
    val genre: String
)

// Lo que devuelvo al cliente
data class BookResponse(
    val id: Long,
    val title: String,
    val authorName: String,
    val genre: String
)
```

**Ventajas:** separación clara de capas, validación independiente, evolución de la API sin afectar el dominio.

### B) Jackson — `@JsonIgnore` y `@JsonProperty`

Controla la serialización directamente sobre la entidad (acoplado, pero rápido):

```kotlin
@Entity
data class User(
    val id: Long,
    val username: String,

    @JsonIgnore           // nunca se serializa
    val passwordHash: String,

    @JsonProperty("full_name")   // cambia el nombre del campo en el JSON
    val fullName: String
)
```

### C) Jackson Custom Serializer

Útil para transformaciones complejas (ej. formatear fechas, enmascarar datos):

```kotlin
class BookSerializer : JsonSerializer<Book>() {
    override fun serialize(book: Book, gen: JsonGenerator, provider: SerializerProvider) {
        gen.writeStartObject()
        gen.writeStringField("id", book.id.toString())
        gen.writeStringField("title", book.title.uppercase())
        gen.writeEndObject()
    }
}

@JsonSerialize(using = BookSerializer::class)
@Entity
data class Book(...)
```

### D) `@JsonView`

Permite definir **vistas** y controlar qué campos se exponen según el contexto (ej. vista pública vs. vista admin):

```kotlin
object Views {
    open class Public
    class Admin : Public()
}

@Entity
data class User(
    @JsonView(Views.Public::class) val id: Long,
    @JsonView(Views.Public::class) val username: String,
    @JsonView(Views.Admin::class)  val email: String      // solo para admins
)

// En el controller:
@JsonView(Views.Public::class)
@GetMapping("/users/{id}")
fun getUser(@PathVariable id: Long) = userService.findById(id)
```

### Comparativa rápida

| Estrategia | Acoplamiento | Flexibilidad | Recomendado para |
|---|---|---|---|
| DTO | Bajo | Alta | Proyectos serios, APIs públicas |
| `@JsonIgnore` | Alto | Baja | Prototipos rápidos |
| Custom Serializer | Medio | Muy alta | Transformaciones complejas |
| `@JsonView` | Medio | Media | Multi-rol sobre misma entidad |

> 📖 [Jackson Annotations — Baeldung](https://www.baeldung.com/jackson-annotations)
> 📖 [Jackson @JsonView](https://www.baeldung.com/jackson-json-view-annotation)

---

## 6. Tests de integración con MockMVC

### Arquitectura del test

```
Cliente
  │
  ▼
Capa REST (HTTP/JSON)
  │
  ▼
Controller
  │
  ▼
Service
  │
  ▼
Repository ──────► Base de datos
                       ├── Producción  → PostgreSQL (servidor)
                       ├── Test local  → H2 in-memory  ó  PostgreSQL (Docker/local)
                       └── CI/CD       → H2 in-memory  ó  PostgreSQL (VM efímera)
```

**MockMVC** actúa como cliente HTTP simulado que prueba el stack completo **desde el nivel JSON**, sin levantar un servidor real.

### Ejemplo completo

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class BookControllerIntegrationTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var bookRepository: BookRepository

    @Test
    @Transactional
    fun `should return book by id`() {
        // Arrange
        val book = bookRepository.save(Book(title = "Clean Code", genre = Genre.DESIGN))

        // Act & Assert
        mockMvc.perform(get("/books/${book.id}"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.title").value("Clean Code"))
            .andExpect(jsonPath("$.genre").value("Diseño"))
    }

    @Test
    fun `should return 404 when book not found`() {
        mockMvc.perform(get("/books/9999"))
            .andExpect(status().isNotFound)
    }
}
```

### H2 para tests

En `src/test/resources/application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect
```

### Consideraciones importantes

- `@Transactional` en el test hace **rollback automático** al terminar cada test — los datos no persisten entre tests.
- Para tests que verifican comportamiento transaccional real, **no uses `@Transactional` en el test** (sino estarías dentro de la misma transacción que el servicio).
- Con H2 hay dialectos ligeramente distintos a PostgreSQL (ej. funciones, tipos). Para máxima fidelidad, usar **Testcontainers** con PostgreSQL real.

### Testcontainers (nivel avanzado)

```kotlin
@Testcontainers
@SpringBootTest
class BookRepositoryTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer<Nothing>("postgres:15")
            .apply { withDatabaseName("testdb") }
    }
}
```

> 📖 [MockMVC — Spring Docs](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#spring-mvc-test-framework)
> 📖 [Testcontainers + Spring Boot](https://www.baeldung.com/spring-boot-testcontainers-integration-test)
> 📖 [Testing Spring Boot Apps — Baeldung](https://www.baeldung.com/spring-boot-testing)

---

## 7. Links útiles

| Recurso | URL |
|---|---|
| Spring Framework Docs | https://docs.spring.io/spring-framework/docs/current/reference/html/ |
| Spring Boot Reference | https://docs.spring.io/spring-boot/docs/current/reference/html/ |
| Baeldung (guías prácticas) | https://www.baeldung.com/spring-tutorial |
| Kotlin + Spring | https://spring.io/guides/tutorials/spring-boot-kotlin/ |
| JPA Buddy (plugin IntelliJ) | https://jpa-buddy.com/ |
| Testcontainers | https://testcontainers.com/ |
| PgAdmin Web | `http://localhost:<port>/browser/` |

---

> **Nota:** estas notas están orientadas a aplicación práctica en el contexto del curso. Para cada tema hay mucho más por explorar — los links son el siguiente paso natural.