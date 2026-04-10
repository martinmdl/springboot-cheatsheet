# Clase 2 — Mapeo Objeto/Relacional (ORM) con foco practico

> Esta version prioriza decisiones de implementacion en Spring Data JPA.

---

## 1. Objetivos de la clase

- Entender el impedance mismatch (objeto vs tabla).
- Modelar identidad (natural vs subrogada).
- Mapear relaciones y herencia con JPA.
- Elegir estrategia de hidratacion (lazy/eager/entity graph).

---

## 2. El problema: dos modelos, reglas distintas

| Mundo objetos | Mundo relacional |
|---|---|
| Grafo de referencias | Tablas + PK/FK |
| Herencia y polimorfismo nativos | No hay herencia nativa |
| Identidad por referencia en memoria | Identidad por clave |
| Metodos y encapsulamiento | Datos + constraints |

Esto genera el impedance mismatch: no hay traduccion 1:1 perfecta.

---

## 3. Identidad: recomendacion para JPA

## Clave natural
- Ejemplos: DNI, CUIT, email.
- Pros: legible para negocio.
- Contras: puede cambiar en el tiempo.

## Clave subrogada (recomendada)
```kotlin
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
var id: Long? = null
```
- Pros: estable, simple para FK.
- Contras: requiere columna extra.

Regla practica en ORM:
- Usa id tecnico para persistencia.
- Valida unicidad de negocio con `@Column(unique = true)` o constraint SQL.

---

## 4. Relaciones: como mapear rapido

### 4.1 Many-to-One (lado con FK)
```kotlin
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "partido_id", nullable = false)
var partido: Partido? = null
```

### 4.2 One-to-Many (lado coleccion)
```kotlin
@OneToMany(mappedBy = "zona", cascade = [CascadeType.ALL], orphanRemoval = true)
var candidatos: MutableList<Candidato> = mutableListOf()
```

### 4.3 Many-to-Many
```kotlin
@ManyToMany
@JoinTable(
  name = "profesor_materia",
  joinColumns = [JoinColumn(name = "profesor_id")],
  inverseJoinColumns = [JoinColumn(name = "materia_id")]
)
var materias: MutableSet<Materia> = mutableSetOf()
```

Tip:
- Si la relacion tiene atributos propios (ej: fechaAlta, prioridad), modelala como entidad intermedia en vez de `@ManyToMany` directo.

---

## 5. Bidireccionalidad: cuando simplifica y cuando complica

Punto clave para esta materia:
- Como el modelo web por request no mantiene objetos vivos entre requests, no hace falta prohibir toda bidireccionalidad.
- Si una referencia bidireccional simplifica mucho la logica, se puede usar.

Cuidado:
- Mantener ambos lados sincronizados en memoria dentro del caso de uso.
- Evitar loops de serializacion JSON con DTOs o anotaciones de Jackson.

---

## 6. Herencia en JPA: matriz de decision

| Estrategia | Pros | Contras | Uso recomendado |
|---|---|---|---|
| `SINGLE_TABLE` | Rapida en lectura, sin joins | Muchas columnas nulas | Jerarquias chicas y alto trafico |
| `JOINED` | Modelo normalizado | Mas joins | Balance general (suele ser buena opcion) |
| `TABLE_PER_CLASS` | Tablas limpias por subclase | UNION para polimorfismo | Casos puntuales |

Ejemplo:
```kotlin
@Inheritance(strategy = InheritanceType.JOINED)
abstract class Partido
```

---

## 7. Hidratacion y fetch strategy

Hidratacion = instanciar objetos de dominio desde datos persistidos.

Opciones:
- `LAZY`: trae bajo demanda.
- `EAGER`: trae siempre.
- `@EntityGraph`: trae relaciones puntuales para un caso de uso.

Ejemplo con EntityGraph:
```kotlin
@EntityGraph(attributePaths = ["candidatos", "candidatos.partido"])
fun findWithCandidatosById(id: Long): Optional<Zona>
```

Regla practica:
- Default `LAZY`.
- Para casos de lectura concretos, usa query dedicada (`EntityGraph` o `JOIN FETCH`).

---

## 8. Donde ejecutar la logica: service vs base

No hay una unica respuesta correcta, depende del criterio:
- Priorizar control y mantenibilidad: hidratar en service y ejecutar logica de dominio ahi.
- Priorizar performance en operaciones masivas: mover parte de logica a SQL/procedures/triggers.

Importante:
- La decision debe ser consciente y documentada por caso de uso.

---

## 9. Errores comunes y antipatrones

- Exponer entidades JPA directo en endpoints.
- Usar `findAll()` sin filtros en tablas grandes.
- EAGER por defecto en todas las relaciones.
- No versionar schema.
- Ignorar N+1 queries.

---

## 10. Fuentes online

- Hibernate User Guide (fetching): https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#fetching
- Spring Data JPA Reference: https://docs.spring.io/spring-data/jpa/reference/
- Many-to-Many en JPA: https://www.baeldung.com/jpa-many-to-many
- Inheritance strategies: https://www.baeldung.com/hibernate-inheritance
