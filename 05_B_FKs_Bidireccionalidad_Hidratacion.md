# Clase 5 Bonus — Anotaciones JPA, FK, hidratacion y decisiones de arquitectura

> Extension de la clase 5 con foco en decisiones practicas para TP/proyecto real.

---

## 1. `@Column`: constraints utiles de verdad

`@Column` te permite alinear invariantes de dominio con restricciones de columna.

```kotlin
@Column(
  name = "email",
  nullable = false,
  unique = true,
  length = 150
)
var email: String = ""
```

Campos importantes:
- `nullable = false`: obliga `NOT NULL` en schema.
- `unique = true`: crea restriccion unica (ojo: para compuestas usa `@Table(uniqueConstraints=...)`).
- `length = 150`: para `VARCHAR`.
- `precision` y `scale`: para decimales (`BigDecimal`).
- `insertable`/`updatable`: controlan si JPA escribe en INSERT/UPDATE.
- `columnDefinition`: SQL custom (usar solo cuando sea realmente necesario).

Ejemplo decimal:
```kotlin
@Column(nullable = false, precision = 12, scale = 2)
var precio: BigDecimal = BigDecimal.ZERO
```

Regla practica:
- Valida tambien en dominio/DTO (Bean Validation), no solo en BD.

---

## 2. `@JoinColumn` y comportamiento real de FK

`@JoinColumn` define la columna FK de una relacion.

```kotlin
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "cliente_id", nullable = false)
var cliente: Cliente? = null
```

Que pasa en la BD:
- La tabla hija guarda `cliente_id`.
- La integridad referencial exige que ese id exista en tabla padre.
- Si borras padre, el resultado depende del `ON DELETE` de la FK.

### Sobre ON DELETE con JPA
JPA estandar no expone todo el control de `ON DELETE` por anotacion portable.
Opciones:
- Definir la FK y su accion en migracion SQL (recomendado).
- Usar extension Hibernate `@OnDelete(action = OnDeleteAction.CASCADE)` en casos puntuales.

Ejemplo SQL (Flyway):
```sql
ALTER TABLE pedido
ADD CONSTRAINT fk_pedido_cliente
FOREIGN KEY (cliente_id)
REFERENCES cliente(id)
ON DELETE RESTRICT;
```

Regla practica:
- Si hijo no tiene sentido sin padre: evaluar `CASCADE`.
- Si queres proteger historial: usar `RESTRICT`.

---

## 3. Hidratar en service y ejecutar logica en ese mismo metodo

Este criterio coincide con tu apunte de clase.

```kotlin
@Service
class PedidoService(
  private val pedidoRepository: PedidoRepository
) {

  @Transactional
  fun confirmarPedido(id: Long) {
    val pedido = pedidoRepository.findById(id)
      .orElseThrow { IllegalArgumentException("Pedido inexistente") }

    pedido.confirmar()   // logica de dominio en el mismo caso de uso
    // dirty checking hace update al commit
  }
}
```

Beneficios:
- Caso de uso transaccional cerrado.
- Menos acoplamiento controller-repository.
- Coherencia de invariantes.

---

## 4. Query methods (Spring Data JPA)

Referencia oficial:
- https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html

Ejemplos:
```kotlin
fun findByActivoTrue(): List<Usuario>
fun findByNombreContainingIgnoreCase(nombre: String): List<Usuario>
fun findByFechaAltaBetween(desde: LocalDate, hasta: LocalDate): List<Usuario>
```

Si el nombre de metodo se vuelve inmanejable:
- Pasar a `@Query`.
- Usar Specifications/Querydsl para filtros dinamicos.

---

## 5. Referencias bidireccionales: no estan prohibidas

Punto clave:
- Como los objetos no viven indefinidamente en memoria entre requests, no hace falta evitar todas las relaciones bidireccionales.
- Si simplifica mucho la logica del caso de uso, se pueden usar.

Cuidados obligatorios:
- Mantener ambos lados sincronizados en metodos helper.
- No exponer entidades directas en JSON (usar DTOs).

Ejemplo helper:
```kotlin
fun agregarItem(item: ItemPedido) {
  items.add(item)
  item.pedido = this
}
```

---

## 6. Isolation levels: rigor de lectura en concurrencia

Permiten elegir balance entre consistencia y throughput.

```kotlin
@Transactional(isolation = Isolation.REPEATABLE_READ)
fun cerrarCaja(...) { ... }
```

Resumen rapido:
- `READ_COMMITTED` (default comun): evita dirty reads.
- `REPEATABLE_READ`: evita non-repeatable reads.
- `SERIALIZABLE`: max aislamiento, menor concurrencia.

No hay nivel "mejor" universal: depende del riesgo funcional del caso de uso.

---

## 7. Triggers en PostgreSQL (pgAdmin / Routines)

Los triggers ejecutan logica automaticamente ante `INSERT/UPDATE/DELETE`.

Flujo tipico en pgAdmin:
1. Database -> Schemas -> public -> Tables -> tu_tabla.
2. Click derecho -> Create -> Trigger.
3. Asociar funcion PL/pgSQL.

Ejemplo minimo:
```sql
CREATE OR REPLACE FUNCTION audit_pedido_fn()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO pedido_audit(pedido_id, fecha, accion)
  VALUES (NEW.id, now(), TG_OP);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_pedido
AFTER INSERT OR UPDATE ON pedido
FOR EACH ROW EXECUTE FUNCTION audit_pedido_fn();
```

---

## 8. Hidratacion BD <-> back y criterio de logica en service vs BD

Hidratacion = proceso de instanciar objetos de dominio desde registros persistidos.

Dos estrategias validas:
- Logica en service (Java/Kotlin):
  - Pros: mayor control, debugging/test unitario mas simple, versionado junto al codigo.
  - Contras: puede requerir mas roundtrips y mover mas datos.
- Logica en BD (SQL/procedure/trigger):
  - Pros: performance en operaciones masivas, menos traslado de datos.
  - Contras: testing/versionado mas delicado, mayor dependencia del motor.

Conclusion practica:
- No hay mejor/peor absoluto.
- La eleccion es por criterio tecnico del caso de uso (volumen, criticidad, mantenibilidad, equipo).

---

## 9. `@DataJpaTest` o `@SpringBootTest`: ambos pueden usar H2

Si, en ambos casos podes usar H2 en tests.

- `@DataJpaTest`:
  - Solo capa de persistencia.
  - Rapido para probar repositorios/queries.
- `@SpringBootTest`:
  - Contexto completo.
  - Ideal para pruebas integrales HTTP + service + repo.

En ambos podes setear profile test con H2:
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
```

---

## 10. Baja/alta logica en PostgreSQL con JPA: si, se puede

Si, JPA puede gestionar baja logica (soft delete) y alta/rehabilitacion.

Patron comun:
- Columna `activo` o `deleted_at`.
- En vez de `DELETE` fisico, hacer `UPDATE`.

Con Hibernate:
```kotlin
@Entity
@SQLDelete(sql = "UPDATE usuario SET activo = false WHERE id = ?")
@Where(clause = "activo = true")
class Usuario(
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  var id: Long? = null,

  @Column(nullable = false)
  var activo: Boolean = true
)
```

Notas:
- `@Where` filtra lecturas normales, pero no todas las consultas nativas/custom.
- Conviene agregar metodos explicitos para incluir inactivos cuando haga falta.

---

## 11. DTOs y mappers en Kotlin: extension function vs companion object

Cuando tenes que convertir entre dominio y DTO, hay dos opciones idiomaticas en Kotlin.

### Opcion 1: extension function (mapper externo)

El mapper vive en un archivo separado, fuera del DTO y fuera del dominio:

```kotlin
// mappers/BookMapper.kt
fun Book.toEditableFieldsDTO(): BookEditableFieldsDTO {
  return BookEditableFieldsDTO(
    type = frontName(),
    title = title,
    description = description,
    coverUrl = coverUrl,
    author = author,
    pages = pages,
    isbn = isbn,
    genre = genre.frontName,
    editorial = editorial,
    language = language.frontName,
    publicationDate = publicationDate,
    state = state.frontName
  )
}

// Uso
val dto = book.toEditableFieldsDTO()
```

Por que no acopla a `Book`:
- Kotlin compila la extension como funcion estatica externa.
- `Book` no importa el DTO ni sabe que existe el mapper.
- La dependencia es unidireccional: `mapper → domain + dto`.

### Opcion 2: companion object (dentro del DTO)

```kotlin
data class BookEditableFieldsDTO(...) {
  companion object {
    fun from(book: Book): BookEditableFieldsDTO {
      return BookEditableFieldsDTO(
        type = book.frontName(),
        title = book.title,
        // ...
        state = book.state.frontName
      )
    }
  }
}

// Uso
val dto = BookEditableFieldsDTO.from(book)
```

Contra: acopla el DTO con el dominio.

### Criterio de eleccion

| Contexto | Opcion recomendada |
|---|---|
| Proyecto chico / prototipo | `companion object` — mas facil de encontrar |
| Arquitectura en capas (clean/hexagonal) | Extension function en `mappers/` |

Regla mental:
- Extension outside → no acopla al receptor.
- Metodo dentro de la clase → si acopla.

Organizacion sugerida:
```
src/
  mappers/
    BookMapper.kt
    UserMapper.kt
```

---

## 12. Flyway en produccion: FAQ practico

### Que problema resuelve

Sin Flyway (`ddl-auto=update`):
- No hay historial confiable de cambios.
- Los entornos (dev / staging / prod) divergen con el tiempo.
- No hay forma de reproducir el estado exacto de la BD.

Con Flyway:
- El schema evoluciona con scripts versionados (`V1`, `V2`, ...).
- Estado reproducible: mismas migraciones → misma BD en cualquier entorno.
- Historial en la tabla `flyway_schema_history`.

### Como funciona al iniciar la app

1. Flyway escanea `resources/db/migration`.
2. Consulta `flyway_schema_history` para ver que versiones ya se aplicaron.
3. Ejecuta en orden solo las pendientes.
4. Registra cada ejecucion (version, checksum, fecha, estado).
5. Luego Hibernate valida el schema (`ddl-auto=validate`).

### Los `Vx` los crea Hibernate automaticamente?

No. Se escriben a mano. Hibernate no genera migraciones versionadas.

Flujo correcto:
1. Cambiás el modelo (`@Entity`).
2. Pensas el impacto SQL.
3. Escribis el script.
4. Flyway lo ejecuta al arrancar.

```sql
-- V3__add_pages.sql
ALTER TABLE book ADD COLUMN pages INT;
```

### Que puedo versionar ademas de estructura?

- Estructura: `CREATE TABLE`, `ALTER TABLE`, `ADD COLUMN`, `DROP COLUMN`.
- Datos controlados: seeds, transformaciones necesarias para que el codigo funcione.
- Indices y constraints.

No se versionan datos dinamicos (clientes, pedidos, etc.).

### Donde veo `flyway_schema_history`?

En la misma BD, con cualquier cliente SQL:

```sql
SELECT * FROM flyway_schema_history ORDER BY installed_rank;
```

Columnas clave: `version`, `description`, `success`, `installed_on`, `checksum`.

### Cuando consultan las versiones en produccion?

1. **Deploy** — verificar que la version esperada se aplico.
2. **Incidente en prod** — detectar desalineacion (`"el codigo espera V14, prod esta en V12"`).
3. **Comparar entornos** — staging vs prod.
4. **Debug de migracion fallida** — Flyway la marca como `failed` y bloquea siguientes ejecuciones.
5. **Auditoria** — cuando se agrego una columna, que cambio se hizo.

### Que pasa si modifico un `Vx` ya ejecutado?

Flyway falla: detecta que el checksum cambió.

Opciones para resolver:
- Opcion correcta: no tocar el script, crear nuevo script correctivo (`V5__fix_x.sql`).
- En desarrollo: borrar la BD y re-ejecutar todo, o correr `flyway repair` para actualizar el checksum.
- En produccion: nunca modificar versiones viejas. Siempre nueva migracion.

Regla clave: un `Vx` es inmutable despues de ejecutarse. Pensalo como un commit ya pusheado.

### Si reconstruyo la BD desde cero?

Flyway ejecuta todas las versiones en orden: `V1 → V2 → ... → Vn`. La BD queda exactamente igual que en cualquier otro entorno.

### Flyway es backup de datos dinamicos?

No. Para eso se usan backups del motor (dump, snapshots, etc.).

### Config recomendada

```yaml
spring:
  jpa:
    ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    out-of-order: true          # util en trabajo en equipo con branches
    baseline-on-migrate: true   # si ya habia una BD sin historial Flyway
```

---

## 13. Checklist de decisiones

- Definir constraints en `@Column` + migracion SQL.
- Explicitar FKs con criterio de borrado (`RESTRICT`/`CASCADE`).
- Mantener la logica principal en service transaccional.
- Usar query methods para casos simples y `@Query` para complejos.
- Elegir isolation por riesgo funcional.
- Documentar cuando una regla vive en trigger/procedure.
- Testear con H2 o PostgreSQL segun fidelidad requerida.

---

## 14. Fuentes online

- JPA `@Column`: https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/column
- JPA `@JoinColumn`: https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/joincolumn
- Spring Data Query Methods: https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
- Spring Transactions + Isolation: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html
- PostgreSQL Triggers: https://www.postgresql.org/docs/current/triggers.html
- Hibernate soft delete (`@SQLDelete`, `@Where`): https://www.baeldung.com/spring-jpa-soft-delete
- Flyway documentation: https://documentation.red-gate.com/fd
