# Clase 3 - Taller Práctico: Mapeo Objeto/Relacional con Spring Boot

> Material de referencia:
> - [Apunte: Mapeo Objetos Relacional](https://docs.google.com/document/d/1YLmp9vMnSzKg2emt3Bx564Tf1CLalShPc98Z8nCoi7s/edit)
> - [Taller ORM - Doc completo](https://docs.google.com/document/d/13vAmPKbWfWpRWze3AhLwnCHfWktfIIXnju3PD_tzyW4/edit)
> - [Backend: eg-politics-springboot-kotlin](https://github.com/uqbar-project/eg-politics-springboot-kotlin)
> - [Frontend: eg-politics-react](https://github.com/uqbar-project/eg-politics-react)
> - Video: [Arquitectura recomendada para Springboot](https://www.youtube.com/watch?v=Rx9y4Dl-NyE)
> - Video: [Mapeo de Herencia en Springboot](https://www.youtube.com/watch?v=9xprRE1Xoco)

---

## 1. El ejemplo: Politics

El dominio del taller es un sistema de elecciones. Las reglas son:

- Cada zona tiene N candidatos. Un candidato pertenece a una zona y a un partido político a la vez.
- Cada candidato puede tener 0 o más promesas (con descripción y fecha).
- Los partidos políticos forman una jerarquía: Peronista (con atributo `populista`) y Preservativo (con `fechaCreacion`). A futuro podría incorporarse una Alianza.

El modelo de objetos incluye relaciones `@OneToMany`, `@ManyToOne` y herencia — exactamente lo que vamos a mapear en esta clase.

---

## 2. Configuración inicial del proyecto

### `application.yml` — parámetros clave

```yaml
spring:
  datasource:
    url: jdbc:postgresql://0.0.0.0:5432/politics
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver
  jpa:
    ddl-auto: create-drop   # destruye y recrea tablas al iniciar — solo para desarrollo
    open-in-view: false     # recomendado (ver sección 6)
    properties:
      hibernate:
        show_sql: true      # útil para ver qué queries genera Hibernate
```

**`ddl-auto` — opciones y cuándo usar cada una:**

| Valor | Comportamiento | Uso |
|---|---|---|
| `create-drop` | Destruye y recrea el schema en cada inicio | Solo desarrollo local |
| `update` | Intenta hacer ALTER TABLE automáticos | Peligroso: sin rollback, puede perder datos |
| `validate` | Verifica que el mapeo coincida con el schema | Producción + Flyway |
| `none` | No hace nada | Producción gestionada externamente |

> No usar `create-drop` ni `update` en el TP ni en producción. La alternativa correcta es `validate` + migraciones con Flyway.

### Levantar la base con Docker

```bash
docker compose up
```

Levanta PostgreSQL en el puerto 5432 y pgAdmin en `http://localhost:5050`. Para resetear la BD desde cero:

```bash
docker compose down --volumes
docker compose up
```

---

## 3. Mapeo de entidades con JPA

### Anotaciones esenciales

```kotlin
@Entity          // marca la clase como entidad persistible
@Id              // campo que es clave primaria
@GeneratedValue  // clave autogenerada por la BD (secuencia o autoincremental)
@Column(length = 150)  // mapea a una columna; permite definir longitud, nullabilidad, etc.
@Transient       // excluye el campo de la persistencia
```

Por defecto, **todos los campos se persisten** a menos que estén marcados como `@Transient`. El nombre de la tabla se infiere del nombre de la clase (se puede sobreescribir con `@Table(name = "...")`).

### Zona

```kotlin
@Entity
class Zona {
    @Id @GeneratedValue
    var id: Long? = null

    @Column(length = 150)
    lateinit var descripcion: String

    @OneToMany(fetch = FetchType.LAZY)
    lateinit var candidates: MutableSet<Candidate>
}
```

Decisiones tomadas:
- **Clave subrogada** (`@GeneratedValue`): la descripción no es una buena clave candidata.
- **`MutableSet`**: la colección no admite repetidos ni requiere orden — coincide con la semántica de relación en el modelo relacional.
- **`FetchType.LAZY`**: no queremos traer todos los candidatos de todas las zonas cuando solo necesitamos llenar un combo. La colección se hidrata solo cuando se accede a ella.
- **Sin cascada**: los candidatos tienen ciclo de vida independiente de la zona. Eliminar una zona no debe eliminar sus candidatos.

### Candidate

```kotlin
@Entity
class Candidate {
    @Id @GeneratedValue
    var id: Long? = null

    @Column(length = 150)
    var nombre = ""

    @ManyToOne
    var partido: Partido? = null

    var votos = 0

    @OneToMany(fetch = FetchType.LAZY, cascade = [CascadeType.ALL])
    @OrderColumn
    var promesas = mutableListOf<Promesa>()
}
```

Decisiones tomadas:
- **`@ManyToOne` sin cascada**: el partido existe independientemente del candidato.
- **`@OneToMany` con `CascadeType.ALL` en promesas**: una promesa no tiene sentido sin su candidato — ciclo de vida compartido. Si se elimina el candidato, se eliminan sus promesas.
- **`@OrderColumn`**: las promesas se guardan en una lista respetando el orden de inserción. Hibernate agrega una columna `promesas_order` en la tabla intermedia para mantener ese orden.
- **`mutableListOf` en lugar de `mutableSetOf`**: se admite orden y eventualmente repetidos.

### Promesa

```kotlin
@Entity
class Promesa(@Column(length = 255) var accionPrometida: String = "") {
    @Id @GeneratedValue
    var id: Long? = null

    @Column
    var fecha: LocalDate = LocalDate.now()
}
```

Sin complejidades especiales: mapeo directo de atributos. El `LocalDate` de Kotlin/Java se mapea a `DATE` en PostgreSQL.

### Partido político — herencia

```kotlin
@Entity
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")
@JsonSubTypes(
    JsonSubTypes.Type(value = Peronista::class, name = "PJ"),
    JsonSubTypes.Type(value = Preservativo::class, name = "PRE")
)
@Inheritance(strategy = InheritanceType.JOINED)
abstract class Partido {
    @Id @GeneratedValue
    var id: Long? = null
    lateinit var nombre: String
    var afiliados: Int = 0
}

@Entity
class Peronista : Partido() {
    var populista = false
}

@Entity
class Preservativo : Partido() {
    var fechaCreacion: LocalDate = LocalDate.now()
}
```

Las anotaciones `@JsonTypeInfo` y `@JsonSubTypes` son de **Jackson** (el serializador JSON de Spring Boot), no de JPA. Le dicen al serializador cómo diferenciar subclases al convertir JSON → objeto: el campo `"type"` en el JSON indica si es `"PJ"` o `"PRE"`.

---

## 4. Repositorios con Spring Data JPA

### Declaración mínima

```kotlin
interface ZonaRepository : CrudRepository<Zona, Long> {
    fun findByDescripcion(descripcion: String): List<Zona>
}
```

`CrudRepository<T, ID>` provee automáticamente `save`, `findById`, `findAll`, `delete`, `count`, etc. Para búsquedas custom, basta con definir el método siguiendo la convención `findByXxx` — Spring Data genera el query SQL automáticamente.

La interfaz genérica recibe:
- `T`: el tipo de la entidad (`Zona`)
- `ID`: el tipo del identificador (`Long`)

### Métodos útiles que hereda `CrudRepository`

```kotlin
repository.save(entidad)           // INSERT si id == null, UPDATE si id != null
repository.findById(id)            // devuelve Optional<T>
repository.findAll()               // devuelve todos los registros
repository.delete(entidad)
repository.count()
```

### Queries custom con `@EntityGraph`

Para evitar `LazyInitializationException` en casos de uso específicos sin cambiar el fetch mode global, se usa `@EntityGraph`:

```kotlin
// En el repositorio — trae zona + sus candidates + el partido de cada candidate
@EntityGraph(attributePaths = ["candidates.partido"])
override fun findById(id: Long): Optional<Zona>

// Para el candidato — trae partido, promesas y opiniones
@EntityGraph(attributePaths = ["partido", "promesas", "opiniones"])
override fun findById(id: Long): Optional<Candidate>
```

Los `attributePaths` usan los nombres de las **propiedades en el modelo de objetos** (no los nombres de las columnas SQL). El resultado es un único JOIN que trae todo en una sola consulta.

---

## 5. Transaccionalidad declarativa

### Manejo manual (JPA puro)

```kotlin
entityManager.transaction.begin()
try {
    entityManager.persist(entidad)
    entityManager.transaction.commit()
} catch (e: PersistenceException) {
    entityManager.transaction.rollback()
} finally {
    entityManager.close()
}
```

Verboso, propenso a errores, pero da control total.

### Manejo declarativo con `@Transactional`

```kotlin
@Service
class CandidateService {

    @Transactional(readOnly = true)   // no abre transacción de escritura; hint para optimizar
    fun getCandidate(id: Long): Candidate = ...

    @Transactional                    // abre transacción; commit si termina bien, rollback si RuntimeException
    fun actualizarCandidate(candidateNuevo: Candidate, id: Long): Candidate { ... }
}
```

**Regla importante:** `@Transactional` solo hace rollback automático ante `RuntimeException` (no chequeadas). Si necesitás rollback ante `Exception` chequeada, hay que configurarlo explícitamente con `@Transactional(rollbackFor = [Exception::class])`.

El ciclo es:
1. Al entrar al método → Spring abre una transacción (o reutiliza una existente según el `TxType`).
2. Si el método termina normalmente → **commit**.
3. Si se lanza una `RuntimeException` → **rollback** automático.

---

## 6. Ciclo de vida de los objetos en JPA

Hibernate categoriza los objetos en tres estados:

**Transient:** el objeto existe en memoria pero nunca fue persistido (o fue creado sin pasar por Hibernate). No tiene sesión asociada.

**Persistent:** el objeto está asociado a una sesión activa. Hibernate lo rastrea — cualquier cambio en sus atributos se detecta automáticamente (*dirty checking*) y se sincroniza con la BD al hacer commit.

**Detached:** el objeto fue persistido pero su sesión ya cerró. Puede tener datos desactualizados. Para sincronizarlo hay que llamar a `merge()` o `save()`.

---

## 7. Sesión, `open-in-view` y el problema N+1

### Qué es la sesión

La sesión de Hibernate (`EntityManager`) es el contexto en el que se resuelven los proxies lazy y se rastrea el estado de los objetos. Es un recurso externo (una conexión del pool al motor de BD) — hay que manejarla con cuidado.

### `open-in-view: true` (default) — por qué es problemático

Con esta configuración la sesión permanece abierta **desde el inicio del request HTTP hasta que el controller termina de serializar la respuesta a JSON**. Esto produce:

1. **El problema N+1:** cuando Jackson serializa una entidad con colecciones lazy, cada acceso a una colección dispara un nuevo query a la BD. Para N zonas con K candidatos cada una → 1 query de zonas + N queries de candidatos + N×K queries de promesas...
2. **Pool de conexiones ocupado:** la conexión queda tomada durante toda la serialización, incluyendo cualquier procesamiento del controller — bajo carga alta, el pool se agota.
3. **Mezcla de responsabilidades:** el controller puede disparar queries a la BD "por accidente" al acceder a relaciones lazy.

### `open-in-view: false` (recomendado)

La sesión dura solo durante la ejecución del método del repositorio (o del service si está anotado con `@Transactional`). Acceder a una colección lazy fuera de ese contexto → `LazyInitializationException`.

```
org.springframework.http.converter.HttpMessageNotWritableException: Could not write JSON:
failed to lazily initialize a collection of role: ar.edu.unsam.politics.domain.Zona.candidatos,
could not initialize proxy - no Session
```

El error parece malo pero en realidad es una señal clara: **te está diciendo que intentaste acceder a datos que no cargaste**. Es un bug que se detecta en desarrollo, no en producción silenciosamente.

### Cómo resolver el problema sin `open-in-view: true`

Hay cuatro estrategias, que se pueden combinar:

**1. DTOs — la más explícita y recomendada**

No expongas entidades JPA directamente en los endpoints. Convertí solo lo que necesita la vista:

```kotlin
data class ZonaPlanaDTO(val id: Long, val descripcion: String)

fun getZonas(): List<ZonaPlanaDTO> =
    zonaRepository.findAll().map { ZonaPlanaDTO(it.id!!, it.descripcion) }
```

Ventaja: no accedés a las colecciones lazy, no hay LazyInitializationException. La sesión cierra, Jackson serializa el DTO sin problemas.

**2. `@EntityGraph` — para casos de uso que necesitan relaciones**

```kotlin
@EntityGraph(attributePaths = ["candidates.partido"])
override fun findById(id: Long): Optional<Zona>
```

Trae todo en un único JOIN dentro de la sesión. El objeto llega completamente hidratado.

**3. `@JsonView` — distintas serializaciones para el mismo objeto**

Define vistas como interfaces anidadas y anota cada campo con qué vistas lo incluyen:

```kotlin
internal class View {
    internal interface Zona {
        interface Detalle
        interface Plana
    }
}

@Entity
class Zona {
    @JsonView(View.Zona.Plana::class, View.Zona.Detalle::class)
    var id: Long? = null

    @JsonView(View.Zona.Plana::class, View.Zona.Detalle::class)
    lateinit var descripcion: String

    @JsonView(View.Zona.Detalle::class)  // solo en la vista de detalle
    lateinit var candidates: MutableSet<Candidate>
}
```

Flexible, pero "ensucia" el objeto de dominio con concerns de serialización.

**4. Eager loading — evitar si es posible**

Cambiar `FetchType.LAZY` a `FetchType.EAGER` resuelve el error pero fuerza a traer las colecciones en *todos* los casos de uso, incluso cuando no se necesitan. Genera exactamente el problema N+1 que intentábamos evitar.

---

## 8. Estrategias de herencia — comparación en código

El taller muestra las tres estrategias con el mismo ejemplo, cambiando una sola anotación:

### JOINED (default del taller)

```kotlin
@Inheritance(strategy = InheritanceType.JOINED)
abstract class Partido
```

Genera: tabla `PARTIDO` (atributos comunes) + tablas `PERONISTA` y `PRESERVATIVO` con solo sus atributos propios. El ID de las subclases es simultáneamente PK y FK hacia `PARTIDO`.

Query resultante: `LEFT JOIN` entre `PARTIDO` y cada subclase.

### TABLE_PER_CLASS

```kotlin
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
abstract class Partido
```

Genera: `PERONISTA` y `PRESERVATIVO` con todos los atributos (incluyendo los heredados). No existe tabla `PARTIDO`.

Query resultante: `UNION` de las subclases con columnas nulas para los atributos que no aplican. No permite FK hacia `PARTIDO` como tipo abstracto.

### SINGLE_TABLE

```kotlin
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "tipo_partido", discriminatorType = DiscriminatorType.INTEGER)
abstract class Partido

@DiscriminatorValue("1")
class Peronista : Partido()

@DiscriminatorValue("2")
class Preservativo : Partido()
```

Genera: una única tabla `PARTIDO` con todos los atributos de todas las subclases. Requiere columna discriminante.

Query resultante: un simple `JOIN` contra una sola tabla, sin UNION ni JOIN adicional.

**El poder del mapeo declarativo:** cambiar de una estrategia a otra es solo cambiar el valor de `@Inheritance(strategy = ...)`. Hibernate maneja el resto.

---

## 9. Serialización JSON — opciones para los controllers

Al diseñar un endpoint hay que decidir qué datos exponer y en qué formato. Las opciones disponibles, de menor a mayor flexibilidad:

**`@JsonIgnore` / `@JsonProperty`:** aplica globalmente a todas las serializaciones de ese objeto. Poco flexible si el mismo campo necesita comportarse diferente en distintos endpoints.

**DTOs específicos por caso de uso:** el approach más explícito. Repetitivo pero claro — si el objeto de dominio cambia, el DTO no cambia automáticamente (puede ser una ventaja de estabilidad).

**`@JsonView`:** diferentes serializaciones para el mismo objeto, activadas por anotación en el controller. El objeto de dominio se "ensucia" un poco con concerns de vista.

**Serializadores custom (`StdSerializer`):** máximo control, pero verbosos. Útil para transformaciones complejas.

**Regla práctica:** para endpoints simples, usar DTOs. Para casos donde el mismo objeto necesita múltiples formas de serialización, `@JsonView`. Evitar exponer entidades JPA directamente — el lazy loading siempre va a generar sorpresas.

---

## 10. Arquitectura del proyecto

```
┌─────────────────────────────────────────────────────┐
│                   React Frontend                     │
│          (axios → JSON over HTTP)                   │
└───────────────────────────┬─────────────────────────┘
                            │ HTTP (REST)
┌───────────────────────────▼─────────────────────────┐
│              @RestController                         │
│   (recibe requests, delega al service, serializa)   │
└───────────────────────────┬─────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────┐
│                  @Service                            │
│  (lógica de aplicación, demarca transacciones)      │
└───────────────────────────┬─────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────┐
│          Repository (CrudRepository)                 │
│        (consultas y actualizaciones a la BD)        │
└───────────────────────────┬─────────────────────────┘
                            │ JDBC / Hibernate
┌───────────────────────────▼─────────────────────────┐
│              PostgreSQL (via Docker)                 │
└─────────────────────────────────────────────────────┘
```

**Responsabilidades:**
- **Controller:** recibe el request HTTP, llama al service, serializa la respuesta. Sin lógica de negocio ni accesos directos a la BD.
- **Service:** orquesta la lógica de aplicación, demarca transacciones con `@Transactional`. Es quien decide qué datos traer y cómo combinarlos.
- **Repository:** solo persiste y consulta. Sin lógica de negocio.
- **Dominio:** las reglas de negocio viven en las entidades (`votar()`, `actualizarPromesas()`, etc.).

---

## 11. Testing de integración

### Con `@SpringBootTest`

Levanta el contexto completo de Spring. Prueba la cadena entera: controller → service → repositorio → BD (H2 en memoria en test).

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class CandidateControllerTest {

    @Autowired lateinit var mockMvc: MockMvc
    @Autowired lateinit var mapper: ObjectMapper
    @Autowired lateinit var repoCandidates: CandidateRepository

    @Test
    @Transactional   // rollback automático al terminar el test
    fun `actualizar la informacion de una persona candidata`() {
        val candidate = repoCandidates.findByNombre("Julio Sosa").get()
        candidate.reset()
        candidate.votar()

        val responseEntity = mockMvc.perform(
            MockMvcRequestBuilders.put("/candidates/${candidate.id}")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(candidate))
        ).andReturn().response

        assertEquals(200, responseEntity.status)

        val actualizado = repoCandidates.findByNombre("Julio Sosa").get()
        assertEquals(1, actualizado.votos)
    }
}
```

**`@Transactional` en el test:** abre una transacción, ejecuta el test, y al finalizar hace **rollback automático**. Esto garantiza que cada test es independiente del anterior: el siguiente test no ve los efectos colaterales del anterior.

### BD de testing — configuración

En `src/test/resources/application.yml` se sobreescribe la configuración para usar H2 en memoria:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password: sa
    driverClassName: org.h2.Driver
  jpa:
    ddl-auto: create-drop
```

---

## 12. Checklist de decisiones de diseño del taller

Al mapear una nueva entidad, responder estas preguntas:

**Identidad:** ¿clave natural o subrogada? Si la clave natural puede cambiar (teléfono, código postal) o es compuesta → subrogada.

**Colección:** ¿`Set` (sin orden, sin duplicados), `List` (con orden, con duplicados), o conjunto ordenado? La respuesta determina si necesitás `@OrderColumn` y cómo se mapea en la BD.

**Fetch mode:** ¿Lazy o Eager? Por defecto Lazy para relaciones `*-to-many`. Solo cambiar a Eager si ese caso de uso siempre necesita la colección y el volumen es acotado.

**Cascada:** ¿el hijo tiene sentido sin el padre? Si no → `CascadeType.ALL` + `orphanRemoval = true`. Si sí (ciclo de vida independiente) → sin cascada.

**Herencia:** `SINGLE_TABLE` si las subclases son similares y querés performance; `JOINED` si querés normalización y soportar relaciones polimórficas con FK; `TABLE_PER_CLASS` si las subclases son muy distintas y nunca necesitás tratar a la superclase como FK.

**Serialización:** ¿qué datos expone cada endpoint? Usar DTOs o `@JsonView`. Nunca exponer entidades JPA directamente si tienen relaciones lazy.

**Transacciones:** lecturas con `@Transactional(readOnly = true)`, escrituras con `@Transactional`. Recordar que solo hace rollback ante `RuntimeException`.