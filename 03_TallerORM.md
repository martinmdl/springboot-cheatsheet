# Clase 3 — Taller ORM en Spring Boot (paso a paso)

> Guia operativa para implementar, probar y evitar errores comunes.

---

## 1. Objetivos del taller

- Configurar JPA/Hibernate para desarrollo.
- Mapear entidades y relaciones con criterio.
- Implementar services transaccionales con hidratacion + logica.
- Probar con `@DataJpaTest` y `@SpringBootTest`.

---

## 2. Setup inicial recomendado

## application.yml (desarrollo)
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/politics
    username: postgres
    password: postgres
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        show_sql: true
```

Por que asi:
- `open-in-view: false` para evitar lazy loading accidental en controller.
- `validate` para no romper schema automaticamente.

---

## 3. Mapeo base de entidades

```kotlin
@Entity
class Candidate(
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  var id: Long? = null,

  @Column(nullable = false, length = 120)
  var nombre: String = "",

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "partido_id", nullable = false)
  var partido: Partido? = null,

  var votos: Int = 0
)
```

Reglas practicas:
- `nullable = false` para invariantes reales.
- `LAZY` por default en asociaciones.
- DTO para salida HTTP (evitar exponer entidad directa).

---

## 4. Repositorios y Query Methods

```kotlin
interface CandidateRepository : JpaRepository<Candidate, Long> {
  fun findByNombreContainingIgnoreCase(nombre: String): List<Candidate>
  fun findByPartidoId(partidoId: Long): List<Candidate>
}
```

Referencia oficial de naming/keywords:
- https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html

Cuando no alcanza un query method:
- Usa `@Query` JPQL/SQL.
- Usa `@EntityGraph` para controlar hidratacion de relaciones.

---

## 5. Service: hidratar entidad y ejecutar logica en el mismo metodo

Este punto es clave de clase:

```kotlin
@Service
class CandidateService(
  private val candidateRepository: CandidateRepository
) {

  @Transactional
  fun registrarVoto(candidateId: Long) {
    // 1) Hidrato desde BD
    val candidato = candidateRepository.findById(candidateId)
      .orElseThrow { IllegalArgumentException("Candidato inexistente") }

    // 2) Ejecuto logica de dominio en el mismo metodo
    candidato.votos += 1

    // 3) No hace falta save explicito si la entidad sigue managed
    // Hibernate sincroniza en commit (dirty checking)
  }
}
```

Ventaja:
- Caso de uso autocontenido y transaccional.

---

## 6. N+1, lazy y serializacion

Sintoma:
- Endpoint lista entidades y cada acceso a relacion lazy dispara query extra.

Soluciones:
- DTOs para respuestas.
- Query dedicada con `JOIN FETCH`.
- `@EntityGraph` en repositorio.

Ejemplo:
```kotlin
@EntityGraph(attributePaths = ["partido", "promesas"])
fun findDetailedById(id: Long): Optional<Candidate>
```

---

## 7. Tests: DataJpaTest vs SpringBootTest (ambos pueden usar H2)

## `@DataJpaTest`
- Levanta solo capa JPA.
- Rapido para probar repositorios y queries.
- Habitualmente usa H2 en memoria.

## `@SpringBootTest`
- Levanta contexto completo.
- Ideal para flujo end-to-end con MockMvc.
- Tambien puede usar H2 en perfil test.

### Ejemplo breve
```kotlin
@DataJpaTest
class CandidateRepositoryTest {
  @Autowired lateinit var repository: CandidateRepository

  @Test
  fun `busca por nombre sin importar mayusculas`() {
    val res = repository.findByNombreContainingIgnoreCase("ana")
    assertThat(res).isNotEmpty
  }
}
```

---

## 8. Mini laboratorio sugerido

1. Crear entidad `Zona` y `Candidate`.
2. Mapear `@OneToMany` + `@ManyToOne`.
3. Implementar `registrarVoto` en service.
4. Exponer endpoint DTO `GET /candidates/{id}`.
5. Agregar test repo (`@DataJpaTest`) y test HTTP (`@SpringBootTest`).

---

## 9. Fuentes online

- Spring Data JPA query methods: https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
- Spring Boot testing: https://docs.spring.io/spring-boot/reference/testing/
- Hibernate dirty checking: https://www.baeldung.com/java-hibernate-entity-dirty-check
- EntityGraph: https://www.baeldung.com/spring-data-jpa-named-entity-graphs
