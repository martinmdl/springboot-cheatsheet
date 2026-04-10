# Clase 5 - ORM: Relaciones JPA, Testing e Integración, Migraciones

> Script original del profesor: [fdodino/scripts-clase - clase3-orm.md](https://github.com/fdodino/scripts-clase/blob/master/phm/ORM/clase3-orm.md)
>
> Repos de referencia:
> - [eg-politics-springboot-kotlin](https://github.com/uqbar-project/eg-politics-springboot-kotlin/tree/update-2026)
> - [eg-profesores-springboot-kotlin](https://github.com/uqbar-project/eg-profesores-springboot-kotlin)
> - [eg-products-springboot-kotlin](https://github.com/uqbar-project/eg-products-springboot-kotlin)

---

## 1. Relaciones JPA / Hibernate

### Por qué Hibernate trabaja con proxies (lo que llamamos "promesas")

Hibernate no carga los objetos relacionados directamente desde la base cuando traés una entidad: en cambio, crea un **proxy** — un objeto sustituto que "promete" devolver el dato real cuando alguien lo necesite. Esto es el corazón del **lazy loading**.

**Cómo funciona internamente:**
1. Hacés `repository.findById(id)` → Hibernate carga la entidad principal con una sola query.
2. Las colecciones o relaciones marcadas como `FetchType.LAZY` se reemplazan por proxies de Hibernate (son subclases generadas en runtime con bibliotecas como ByteBuddy o javassist).
3. Cuando el código accede por primera vez a esa colección (e.g., `candidato.promesas`), el proxy intercepta la llamada y dispara la query a la BD.
4. Esto solo funciona si hay una **sesión de Hibernate abierta** en ese momento — de ahí la importancia de `open-in-view` y las transacciones.

> **Implicancia práctica:** si intentás acceder a una relación lazy *fuera* de una transacción o sesión abierta, obtenés `LazyInitializationException`. El proxy existe pero no puede resolver la promesa.

**Referencia:** [Hibernate ORM - Lazy Loading](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#fetching-strategies)

---

### `orphanRemoval = true`

Cuando tenés una relación `@OneToMany` y eliminás un elemento de la colección en Java/Kotlin (sin tocar la BD directamente), Hibernate por defecto **desvincula** ese objeto pero no lo borra de la tabla. El registro queda "huérfano" — sin ninguna FK que apunte a él.

Con `orphanRemoval = true`, Hibernate detecta que el objeto ya no pertenece a ningún padre y emite un `DELETE` automáticamente.

```kotlin
@OneToMany(cascade = [CascadeType.ALL], fetch = FetchType.LAZY, orphanRemoval = true)
@OrderColumn
var promesas = mutableListOf<Promesa>()
```

**Diferencia con `CascadeType.REMOVE`:** `CascadeType.REMOVE` solo aplica cuando borrás el padre (`DELETE` en cascada). `orphanRemoval` aplica cuando el hijo se *desvincula* de la colección del padre (aunque el padre siga existiendo).

> **Para el TP:** si tu entidad tiene colecciones de objetos "dependientes" (sin sentido fuera del padre), usá `orphanRemoval = true`. Ejemplo clásico: líneas de pedido, ítems de una factura.

---

### `@JoinColumn` en relaciones `@OneToMany`

Por defecto, JPA mapea una relación `@OneToMany` con una **tabla intermedia** (join table). Para evitarla y usar directamente una FK en la tabla "hija", agregás `@JoinColumn`:

```kotlin
@OneToMany(cascade = [CascadeType.ALL], fetch = FetchType.LAZY)
@JoinColumn(name = "candidato_id")  // FK en la tabla `promesa`
var promesas = mutableListOf<Promesa>()
```

Esto genera en la tabla `promesa` una columna `candidato_id` que apunta al padre. Más limpio, más eficiente — una query menos para las operaciones de escritura.

**Cuándo usar tabla intermedia vs FK directa:**
- FK directa (`@JoinColumn`): cuando el hijo no puede existir sin el padre (relación de composición).
- Tabla intermedia (default o `@JoinTable`): en relaciones `@ManyToMany` o cuando ambos lados pueden existir independientemente.

---

### Relaciones `@ManyToMany` y el problema de la bidireccionalidad

En el modelo de objetos elegimos deliberadamente **no tener referencias bidireccionales** (e.g., `Materia` no conoce a sus `Profesor`es). Las razones:

1. **Costo de mantenimiento:** si `Profesor` agrega a su lista de materias y `Materia` agrega a su lista de profesores, tenés que sincronizar ambos lados manualmente o entrás en loops infinitos.
2. **Riesgo de stack overflow** en serialización JSON si no configurás `@JsonManagedReference` / `@JsonBackReference` o `@JsonIgnore`.
3. **Complejidad innecesaria** cuando el dominio no lo requiere.

**Cómo resolver la bidireccionalidad en la respuesta HTTP sin tenerla en el modelo:**

El modelo relacional *sí* tiene esa bidireccionalidad (una tabla intermedia `profesor_materia`), así que podés hacer un JOIN partiendo desde cualquier lado usando JPQL:

```kotlin
@Query("""
    SELECT m.id as id,
           m.nombre as nombre,
           m.anio as anio,
           p.id as profesorId,
           p.nombreCompleto as profesorNombre
      FROM Profesor p
           INNER JOIN p.materias m
     WHERE m.id = :id
""")
fun findFullById(id: Long): List<MateriaFullRowDTO>
```

El resultado es un **producto cartesiano** (N filas = 1 materia × N profesores). El service agrupa esas filas en un DTO:

```kotlin
val materia = materiasDTO.first()
val profesores = materiasDTO.map { ProfesorDTO(it.getProfesorId(), it.getProfesorNombre()) }
return MateriaDTO(materia.getId(), materia.getNombreLindo(), materia.getAnio(), profesores)
```

**Referencias:**
- [Pros and cons of JPA bidirectional relationships - Stack Overflow](https://stackoverflow.com/questions/22461613/pros-and-cons-of-jpa-bidirectional-relationships)
- [ManyToMany en JPA - Baeldung](https://www.baeldung.com/jpa-many-to-many)

---

### Interface DTO para queries custom (`Projection`)

Cuando hacés una query JPQL que devuelve columnas custom (no una entidad completa), JPA necesita saber cómo mapear esas columnas. La solución es una **interface projection**:

```kotlin
interface MateriaFullRowDTO {
    fun getId(): Long
    @Value("#{target.nombre}")   // renombra el campo del query
    fun getNombreLindo(): String
    fun getAnio(): Int
    fun getProfesorId(): Long
    fun getProfesorNombre(): String
}
```

- Los **nombres de los getters** deben coincidir (en camelCase) con los **alias** del query (`as profesorId` → `getProfesorId()`).
- `@Value("#{target.campo}")` permite mapear con un nombre diferente.
- Spring Data crea la implementación de la interfaz en runtime — no necesitás clase concreta.

**Alternativas:** también podés usar `@SqlResultSetMapping` con clases concretas o record classes de Kotlin para queries más complejas.

**Referencia:** [Spring Data JPA Projections - Baeldung](https://www.baeldung.com/spring-data-jpa-projections)

---

### `@Transactional` y sus variantes

```kotlin
@Transactional(Transactional.TxType.NEVER)
fun getMateria(id: Long): MateriaDTO { ... }
```

`TxType.NEVER` significa: "este método **no debe** ejecutarse dentro de una transacción activa; si hay una, lanzá excepción". Se usa cuando querés asegurarte de que el método no participa en ninguna transacción (útil para operaciones de solo lectura fuera del contexto transaccional).

**Tabla resumen de TxType:**

| TxType | Comportamiento |
|---|---|
| `REQUIRED` (default) | Usa la transacción existente o crea una nueva |
| `REQUIRES_NEW` | Siempre crea una transacción nueva, suspende la actual |
| `SUPPORTS` | Usa la existente si hay, si no ejecuta sin transacción |
| `NOT_SUPPORTED` | Siempre ejecuta sin transacción, suspende la actual |
| `MANDATORY` | Requiere una transacción existente; si no hay, lanza excepción |
| `NEVER` | Requiere que NO haya transacción; si hay, lanza excepción |

Para lecturas simples en el service, preferí `@Transactional(readOnly = true)` — es una pista a Hibernate para optimizar (no trackea dirty checking).

**Referencia:** [Spring @Transactional - Baeldung](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring)

---

### El problema N+1 y cómo evitarlo

Si tenés `FetchType.LAZY` y en un loop accedés a la colección de cada entidad, Hibernate dispara **1 query para el listado + N queries para cada colección**. Eso es el problema N+1.

```kotlin
// MAL: dispara 1 query para todos los productos + N queries para los fabricantes
val productos = productoRepository.findAll()
productos.forEach { println(it.fabricante.nombre) }  // lazy → 1 query por producto
```

**Soluciones:**
- Usar `JOIN FETCH` en JPQL para traer todo en una sola query.
- Usar `@EntityGraph` para indicar qué relaciones traer eager en un query específico.
- Para el TP: no usar `getAll()` si después vas a navegar relaciones — hacé una query custom que traiga solo lo que necesitás.

**Referencia:** [N+1 problem in Hibernate - Vlad Mihalcea](https://vladmihalcea.com/n-plus-1-query-problem/)

---

## 2. Sesión de Hibernate y `open-in-view`

### Qué es la sesión de Hibernate

La **sesión de Hibernate** (`Session` / `EntityManager`) es el contexto en el cual se resuelven los proxies. Tiene un **caché de primer nivel**: si pedís la misma entidad dos veces dentro de la misma sesión, la segunda vez la devuelve del caché sin ir a la BD.

### `open-in-view: true` (default en Spring Boot) — **NO recomendado para producción**

Con esta configuración, Spring mantiene la sesión de Hibernate **abierta desde el inicio del request HTTP hasta que se envía la respuesta** (incluyendo la serialización JSON).

**Problema:**
- La conexión a la BD queda ocupada durante toda la renderización de la vista/respuesta.
- Bajo carga alta, el pool de conexiones se agota: otros requests quedan bloqueados esperando una conexión.
- Permite acceder a relaciones lazy "por accidente" desde el controller o el serializer JSON — esto parece conveniente pero es mala práctica (el controller no debería disparar queries a la BD).

### `open-in-view: false` (recomendado para el TP y producción)

```yaml
spring:
  jpa:
    open-in-view: false
```

Con esto, la sesión solo está abierta durante la ejecución del método del **repositorio** (o del service si está anotado con `@Transactional`). Si intentás acceder a una relación lazy fuera de ese contexto → `LazyInitializationException`.

**Beneficio:** te obliga a ser explícito sobre qué datos traés. Si el service cierra su transacción y el controller intenta navegar una relación lazy, falla rápido y ruidosamente — es un bug que se detecta en desarrollo, no en producción.

**Configuración recomendada para el TP:**
```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        default_batch_fetch_size: 16  # optimiza lazy loading en colecciones
```

---

## 3. Testing de integración

### Opciones de anotaciones

| Anotación | Qué levanta | Cuándo usarla |
|---|---|---|
| `@SpringBootTest` | Contexto completo de Spring (todo) | Tests end-to-end; cubre controller → service → repo |
| `@DataJpaTest` | Solo la capa JPA (repositorios + H2 en memoria) | Testear queries custom o comportamiento del repo |
| `@WebMvcTest` | Solo el controller (mockea el service) | Testear lógica del controller aislada (poco útil si el controller es thin) |

### `@DataJpaTest` — para queries complejos

Útil para probar el método `findFullById` con casos de equivalencia (materia sin profesores, materia con varios profesores):

```kotlin
@DataJpaTest
class MateriaRepositoryTest {

    @Autowired lateinit var materiaRepository: MateriaRepository
    @Autowired lateinit var profesorRepository: ProfesorRepository

    @Test
    fun `materia con dos profesores devuelve producto cartesiano`() {
        val materia = materiaRepository.save(Materia(nombre = "Algoritmos II", anio = 1))
        profesorRepository.save(Profesor(nombreCompleto = "Fernando Dodino").apply { materias = mutableSetOf(materia) })
        profesorRepository.save(Profesor(nombreCompleto = "Julián Mosquera").apply { materias = mutableSetOf(materia) })

        val resultado = materiaRepository.findFullById(materia.id)

        assertEquals(2, resultado.size)
    }
}
```

- Automáticamente usa H2 en memoria.
- Cada test se ejecuta en una transacción que se revierte al final → sin efectos colaterales entre tests.

### `@SpringBootTest` — test de integración completo

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class ProfesorControllerTest {
    @Autowired lateinit var mockMvc: MockMvc
    @Autowired lateinit var mapper: ObjectMapper
    
    @Test
    fun `GET de un profesor trae sus materias`() {
        mockMvc.perform(MockMvcRequestBuilders.get("/profesores/1"))
            .andExpect(MockMvcResultMatchers.status().isOk)
            .andExpect(MockMvcResultMatchers.jsonPath("$.materias").isArray)
    }
}
```

**ObjectMapper vs JsonPath:**
- **ObjectMapper:** deserializa el JSON a un objeto Kotlin/Java; útil si querés comparar contra el objeto de dominio.
- **JsonPath (`$.campo`):** navega el JSON directamente; más resiliente si el DTO no coincide exactamente con el objeto de dominio. Preferible cuando el endpoint devuelve un DTO diferente a la entidad.

**`@Transactional` en tests:** anotar el test con `@Transactional` hace que todos los cambios se reviertan al terminar. Ideal para tests que producen efectos (PUT, POST).

**Configuración de BD para tests** — dos opciones:
1. Archivo `src/test/resources/application.yml` con la config H2 (sobreescribe el application.yml principal).
2. Anotación `@ActiveProfiles("test")` + archivo `application-test.yml`.

**Referencias:**
- [Spring Boot Testing - Baeldung](https://www.baeldung.com/spring-boot-testing)
- [DataJpaTest - reflectoring.io](https://reflectoring.io/spring-boot-data-jpa-test/)
- [Unit test vs Integration test - testim.io](https://www.testim.io/blog/unit-test-vs-integration-test/)

---

## 4. Migraciones con Flyway

### Por qué migraciones

El valor de `ddl-auto` en `application.yml` determina qué hace Hibernate con el schema al arrancar:

| Valor | Comportamiento | Cuándo usarlo |
|---|---|---|
| `create-drop` | Crea el schema al iniciar, lo borra al cerrar | Solo desarrollo local |
| `update` | Intenta hacer ALTER TABLE automáticos | **Peligroso:** no tiene rollback, puede perder datos |
| `validate` | Verifica que el mapeo coincida con el schema real | Producción (junto con migraciones) |
| `none` | No hace nada sobre el schema | Producción gestionada externamente |

`update` parece cómodo pero no guarda historial, no es reproducible y puede fallar silenciosamente. **Para el TP: `validate` + Flyway.**

### Cómo funciona Flyway internamente

Flyway mantiene una tabla `flyway_schema_history` en tu BD con el registro de cada script ejecutado (versión, nombre, checksum, fecha, éxito/fallo). Al arrancar la app:

1. Lee todos los scripts en `src/main/resources/db/migration/`.
2. Compara con lo que ya ejecutó (via `flyway_schema_history`).
3. Ejecuta **en orden** solo los scripts nuevos.
4. Si un script ya ejecutado fue modificado (el checksum cambió), **falla** — esto te protege de cambios accidentales en el historial.

### Nomenclatura de scripts

```
db/migration/
├── V1__initial_creation.sql       # Versión: se ejecuta y queda registrada
├── V2__add_fabricante.sql         # Versión siguiente
├── U2__add_fabricante.sql         # Undo de V2 (necesita Flyway Teams)
```

- `V{número}__{descripcion}.sql` → script de versión, se ejecuta una vez.
- `U{número}__{descripcion}.sql` → script de undo (revertir la versión correspondiente).
- El doble guión bajo `__` es el separador entre número y descripción.

**Formato de versión con timestamp** (recomendado para trabajo en equipo):
```
V20261003153320__add_fabricante.sql   # AAAAMMDDHHMMSS
```

Esto evita conflictos cuando dos personas crean migraciones en paralelo.

### Configuración en `application.yml`

```yaml
spring:
  jpa:
    ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    out-of-order: true          # permite ejecutar V4 si falta V3 (útil en branches paralelas)
    baseline-on-migrate: true   # si ya existe una BD sin historial Flyway, establece baseline
    baseline-version: 1         # desde qué versión empieza el baseline
```

**`out-of-order: true`:** permite que si alguien commitió una V4 antes que vos commiteás tu V3, Flyway ejecute V3 igual sin romper. Útil en desarrollo en equipo.

**`baseline-on-migrate`:** si ya tenés una BD creada "a mano" y querés empezar a usar Flyway, esta opción le dice a Flyway "asumí que todo lo anterior ya está aplicado hasta `baseline-version`" y seguí desde ahí.

### sqitch — alternativa a Flyway

[sqitch](https://sqitch.org/) maneja tres tipos de scripts por cambio:
- `deploy/` → equivalente a `Vn` (aplicar el cambio)
- `revert/` → equivalente a `Un` (deshacer el cambio)
- `verify/` → script que verifica que el deploy funcionó (¡esto es lo que lo diferencia!)

La verificación automática es su principal ventaja sobre Flyway: podés correr `sqitch verify` para confirmar que el estado de la BD es el esperado.

**Referencia:** [Flyway documentation](https://documentation.red-gate.com/fd) | [sqitch](https://sqitch.org/)

---

## 5. Consultas custom y cómo evitar `getAll()`

`repository.findAll()` carga **todas las filas** de la tabla. Si tenés 100.000 productos, eso es un problema.

**Alternativas:**
- Paginación con `Pageable`: `repository.findAll(PageRequest.of(0, 20))`.
- Query con filtros: `repository.findByNombreContaining(nombre)`.
- Proyecciones (interfaces DTO) para traer solo los campos que necesitás.

```kotlin
// En el repository
fun findByNombreContaining(nombre: String): List<ProductoDTO>

// Query custom con proyección
@Query("SELECT p.id as id, p.nombre as nombre FROM Producto p WHERE p.activo = true")
fun findActivosResumen(): List<ProductoResumenDTO>
```

---

## 6. DTOs y extensiones en Kotlin

En Kotlin podés definir `fromJson` y `toJson` como **extension functions** del DTO, evitando polución en las clases de dominio:

```kotlin
// Extension function en el DTO
fun ProductoDTO.toDomain(): Producto = Producto(
    nombre = this.nombre,
    precio = this.precio
)

fun Producto.toDTO(): ProductoDTO = ProductoDTO(
    id = this.id,
    nombre = this.nombre
)
```

Esto mantiene la lógica de conversión en la capa de presentación sin mezclarla con el dominio.

---

## Resumen de configuración recomendada para el TP

```yaml
spring:
  jpa:
    open-in-view: false          # no mantener sesión abierta durante el request
    ddl-auto: validate           # validar schema, no modificarlo
    properties:
      hibernate:
        default_batch_fetch_size: 16
  flyway:
    enabled: true
    out-of-order: true
    baseline-on-migrate: true
```

**Checklist:**
- [ ] Usar `open-in-view: false` y resolver relaciones lazy dentro de transacciones en el service.
- [ ] Definir DTOs para las respuestas HTTP; no exponer entidades JPA directamente.
- [ ] Usar `orphanRemoval = true` en relaciones de composición.
- [ ] Evitar `findAll()` sin filtros; usar queries custom o paginación.
- [ ] Scripts de migración con nombre versionado (timestamp o número incremental).
- [ ] Tests de integración con `@SpringBootTest` + H2 en `src/test/resources/application.yml`.
- [ ] Anotar tests con efectos con `@Transactional` para rollback automático.