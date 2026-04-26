# Guía: BD con Spring Boot — Postgres en prod, H2 en tests, Initializer idempotente

> Referencias: [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/) | [H2 Database](https://h2database.com/html/main.html) | [Spring Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles)

---

## Índice

1. [Estructura de archivos](#1-estructura-de-archivos)
2. [Configuración — application.properties](#2-configuración--applicationproperties)
3. [Docker — Postgres en producción/desarrollo](#3-docker--postgres-en-produccióndesarrollo)
4. [El Initializer — cómo funciona y cómo hacerlo idempotente](#4-el-initializer--cómo-funciona-y-cómo-hacerlo-idempotente)
5. [H2 para tests — configuración y comportamiento](#5-h2-para-tests--configuración-y-comportamiento)
6. [Cómo testear: @SpringBootTest vs @DataJpaTest](#6-cómo-testear-springboottest-vs-datajpatest)
7. [Cómo verificar que JPA está conectado a Postgres](#7-cómo-verificar-que-jpa-está-conectado-a-postgres)
8. [Checklist](#8-checklist)

---

## 1. Estructura de archivos

```
src/
├── main/
│   ├── kotlin/
│   │   └── ...
│   │       └── Initializer.kt          ← bootstrap de datos
│   └── resources/
│       └── application.properties      ← config de producción/desarrollo
└── test/
    ├── kotlin/
    │   └── ...
    └── resources/
        └── application-test.properties ← config exclusiva de tests (H2)
```

> `application-test.properties` va en `test/resources`, no en `main/resources`. Spring carga primero `main/resources` y luego sobreescribe con `test/resources` cuando corre tests.

---

## 2. Configuración — application.properties

### `main/resources/application.properties` (Postgres)

```properties
# Conexión a Postgres (via Docker)
spring.datasource.url=jdbc:postgresql://localhost:5432/mi_db
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA / Hibernate
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

> `ddl-auto=update` para el primer run: Hibernate crea las tablas automáticamente a partir de las entidades. En runs subsiguientes, solo aplica cambios de schema. Los datos persisten mientras el volumen de Docker esté activo.

### `test/resources/application-test.properties` (H2)

```properties
# H2 en memoria
spring.datasource.url=jdbc:h2:mem:testdb;MODE=PostgreSQL
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# JPA / Hibernate
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.h2.console.enabled=true
```

> `MODE=PostgreSQL` reduce diferencias de dialecto entre H2 y Postgres. No las elimina, pero evita los errores más comunes.

### Dependencias en `build.gradle.kts`

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")           // driver Postgres
    testImplementation("com.h2database:h2")            // H2 solo en tests
}
```

---

## 3. Docker — Postgres en producción/desarrollo

### `docker-compose.yml` (en la raíz del proyecto)

```yaml
services:
  db:
    image: postgres:16-alpine
    container_name: mi_proyecto_db
    environment:
      POSTGRES_DB: mi_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data   # los datos sobreviven al reinicio

volumes:
  postgres_data:
```

```bash
docker compose up -d          # levantar en background
docker compose down           # detener (datos se conservan)
docker compose down --volumes # detener Y borrar datos (reset total)
```

> El volumen `postgres_data` es lo que hace que los datos sobrevivan entre reinicios del contenedor. Si lo eliminás, la BD vuelve a estar vacía y el Initializer insertará todo de nuevo.

---

## 4. El Initializer — cómo funciona y cómo hacerlo idempotente

### Cómo funciona

```kotlin
@Component
class Initializer(
    private val rolRepository: RolRepository,
    private val usuarioRepository: UsuarioRepository
) {
    @PostConstruct
    fun init() {
        // se ejecuta una vez al arrancar el contexto de Spring
        // después de que las dependencias están inyectadas
    }
}
```

**Flujo de arranque:**
1. Spring crea el contexto.
2. Instancia los beans (`@Component`).
3. Inyecta dependencias.
4. Ejecuta métodos `@PostConstruct` → acá corre el Initializer.

El Initializer corre **en cada arranque de la app**, sin importar `ddl-auto`. Con `update` y un volumen persistente, los datos ya existen en la BD — por eso hay que hacerlo idempotente.

### Idempotencia — tres patrones

**Patrón 1 — chequeo previo (el más simple):**

```kotlin
@PostConstruct
fun init() {
    if (!rolRepository.existsByNombre("ADMIN")) {
        rolRepository.save(Rol(nombre = "ADMIN"))
    }
    if (!usuarioRepository.existsByEmail("admin@demo.com")) {
        usuarioRepository.save(Usuario(email = "admin@demo.com", rol = rolAdmin))
    }
}
```

Requiere definir `existsByNombre` y `existsByEmail` en los repositorios (Spring Data los genera automáticamente por convención de nombres).

**Patrón 2 — guardar solo si la BD está vacía (para datos de seed globales):**

```kotlin
@PostConstruct
fun init() {
    if (rolRepository.count() == 0L) {
        rolRepository.saveAll(listOf(
            Rol(nombre = "ADMIN"),
            Rol(nombre = "USER")
        ))
    }
}
```

Más simple pero menos granular — si se agrega un rol nuevo después, no lo inserta.

**Patrón 3 — constraints únicos en la BD + ignorar excepción:**

Definir `UNIQUE` en la entidad:

```kotlin
@Entity
@Table(uniqueConstraints = [UniqueConstraint(columnNames = ["nombre"])])
class Rol(val nombre: String) {
    @Id @GeneratedValue
    var id: Long? = null
}
```

La BD rechaza duplicados automáticamente. Si el Initializer intenta insertar uno que ya existe, la BD lanza excepción. Se puede ignorar con un bloque `try/catch` por inserción, pero el Patrón 1 es más limpio.

> **Recomendación:** Patrón 1 + constraints únicos como red de seguridad. El chequeo previo evita el error; el constraint único garantiza consistencia aunque el chequeo falle.

---

## 5. H2 para tests — configuración y comportamiento

H2 es una BD relacional **que corre en memoria dentro del proceso de la app**. No necesita Docker ni procesos externos. Spring Boot la levanta automáticamente cuando está en el classpath y el perfil de test activo.

### Activar con `@ActiveProfiles("test")`

```kotlin
@SpringBootTest
@ActiveProfiles("test")
class MiTest {
    // Spring carga application-test.properties
    // datasource → H2 en memoria
    // Initializer → corre normalmente (contexto completo)
}
```

Con `ddl-auto=create-drop` en el perfil test, la BD se crea desde cero al arrancar el contexto y se destruye al cerrarlo. **No hay duplicados** porque la BD es efímera.

### ¿El Initializer corre con H2?

Depende de cómo testees:

| Anotación | ¿Carga el Initializer? |
|---|---|
| `@SpringBootTest` + `@ActiveProfiles("test")` | ✅ Sí — contexto completo |
| `@DataJpaTest` | ❌ No — solo carga capa JPA |
| `@DataJpaTest` + `@Import(Initializer::class)` | ✅ Sí — lo importás explícitamente |

### Diferencias H2 vs Postgres a tener en cuenta

H2 no implementa todas las extensiones de Postgres. Con `MODE=PostgreSQL` se reducen, pero pueden aparecer incompatibilidades con:
- Tipos específicos: `jsonb`, `uuid`, `array`
- SQL nativo (`@Query(nativeQuery = true)`) con funciones de Postgres
- `ON CONFLICT DO NOTHING`

Si usás alguna de estas features, los tests en H2 pueden pasar y fallar en producción. La alternativa es **Testcontainers** (Postgres real en Docker para tests), pero eso ya es otro nivel de complejidad.

---

## 6. Cómo testear: `@SpringBootTest` vs `@DataJpaTest`

| | `@SpringBootTest` | `@DataJpaTest` |
|---|---|---|
| Qué carga | Contexto completo | Solo capa JPA |
| Velocidad | Lento | Rápido |
| Initializer | Corre | No corre (salvo `@Import`) |
| Datasource | El configurado (o H2 con perfil) | H2 automáticamente |
| Cuándo usarlo | Tests de integración end-to-end | Tests de repositorios y queries |

```kotlin
// Test de integración completo
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class CandidatoControllerTest { ... }

// Test de repositorio aislado
@DataJpaTest
class CandidatoRepositoryTest {
    @Autowired lateinit var repo: CandidatoRepository

    @Test
    fun `findByNombre devuelve el candidato correcto`() { ... }
}
```

> No son excluyentes. En un proyecto real usás ambos: `@DataJpaTest` para probar queries custom y `@SpringBootTest` para los flujos completos.

---

## 7. Cómo verificar que JPA está conectado a Postgres

Al iniciar la app, en los logs aparece:

```
HikariPool-1 - Starting...
HikariPool-1 - Added connection org.postgresql.jdbc.PgConnection
HHH000400: Using dialect: PostgreSQLDialect
```

Si algo falla, los errores típicos son:

| Error | Causa probable |
|---|---|
| `Connection refused` | Docker no está corriendo o el puerto no coincide |
| `Password authentication failed` | Credenciales distintas entre `docker-compose.yml` y `application.properties` |
| `Table doesn't exist` | `ddl-auto=validate` o `none` sin haber creado las tablas |
| `LazyInitializationException` | Acceso a relación lazy fuera de transacción (`open-in-view: false`) |

---

## 8. Checklist

**Postgres + Docker:**
- [ ] `docker-compose.yml` con volumen persistente.
- [ ] Credenciales iguales en `docker-compose.yml` y `application.properties`.
- [ ] Driver `runtimeOnly("org.postgresql:postgresql")` en `build.gradle.kts`.
- [ ] `ddl-auto=update` para el primer run (Hibernate crea las tablas).

**Initializer:**
- [ ] Anotado con `@Component` y `@PostConstruct`.
- [ ] Idempotente: chequeo previo con `existsBy...` antes de cada `save()`.
- [ ] Constraints únicos en las entidades como red de seguridad.

**H2 para tests:**
- [ ] `testImplementation("com.h2database:h2")` en `build.gradle.kts`.
- [ ] `application-test.properties` en `src/test/resources/` (no en `main`).
- [ ] `ddl-auto=create-drop` en el perfil test.
- [ ] `MODE=PostgreSQL` en la URL de H2.
- [ ] Tests con `@ActiveProfiles("test")` o `@DataJpaTest`.