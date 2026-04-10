# 0. Arquitectura y Tips de un Backend en Objetos

> Material de referencia:
> - [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
> - [Martin Fowler - DTO](https://martinfowler.com/eaaCatalog/dataTransferObject.html)
> - [Martin Fowler - Service Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)
> - [Martin Fowler - Repository](https://martinfowler.com/eaaCatalog/repository.html)
> - Video: [Arquitectura recomendada para Springboot](https://www.youtube.com/watch?v=Rx9y4Dl-NyE)

---

## 1. La arquitectura en capas

Un backend Spring Boot bien estructurado separa responsabilidades en capas. Cada capa tiene una única razón para cambiar y solo se comunica con la que tiene inmediatamente debajo.

```
┌─────────────────────────────────────────────┐
│             Cliente (React, etc.)            │
│              JSON over HTTP                  │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│              @RestController                 │
│  - Recibe el request HTTP                   │
│  - Valida parámetros de entrada             │
│  - Llama al Service                         │
│  - Convierte dominio → DTO con el Mapper    │
│  - Responde con el status HTTP correcto     │
└─────────────────┬───────────────────────────┘
                  │ objetos de dominio
┌─────────────────▼───────────────────────────┐
│                 @Service                     │
│  - Orquesta la lógica de aplicación         │
│  - Demarca transacciones (@Transactional)   │
│  - Opera sobre objetos de dominio puros     │
│  - No sabe que existe el DTO                │
└─────────────────┬───────────────────────────┘
                  │ entidades JPA
┌─────────────────▼───────────────────────────┐
│            Repository / DAO                  │
│  - Persiste y consulta la BD                │
│  - Sin lógica de negocio                    │
│  - CrudRepository / JpaRepository           │
└─────────────────┬───────────────────────────┘
                  │ JDBC / Hibernate
┌─────────────────▼───────────────────────────┐
│           Base de datos (PostgreSQL)         │
└─────────────────────────────────────────────┘
```

**La regla de dependencia:** cada capa solo conoce a la que tiene debajo. El Controller conoce al Service. El Service conoce al Repository. Nadie mira hacia arriba.

---

## 2. DTO — Data Transfer Object

### Qué es y para qué sirve

Un **DTO** (Data Transfer Object) es un objeto cuya única responsabilidad es transportar datos entre capas o sistemas. No tiene comportamiento — solo campos y, a veces, validaciones de formato.

El problema que resuelve es doble:

**Seguridad:** tu entidad `Empleado` tiene nombre, DNI, legajo, salario, historial de liquidaciones, área y cargo. En el listado de nómina, ¿necesitás exponer el salario de cada persona? ¿El historial de liquidaciones? Exponer la entidad directamente significa exponer datos que nunca deberían viajar en esa respuesta.

**Performance:** traer del cliente solo los campos que va a usar. No serializar colecciones lazy innecesariamente. No generar el problema N+1 por acceder a relaciones que nadie pidió.

### Un DTO por caso de uso, no por entidad

No existe un único DTO de `Empleado`. Existen tantos DTOs como formas distintas de necesitar esos datos:

```kotlin
// Para el listado de nómina — solo lo necesario
data class EmpleadoResumenDTO(
    val id: Long,
    val nombre: String,
    val cargo: String,
    val area: String
)

// Para el detalle de un empleado — todo, con controles de acceso
data class EmpleadoDetalleDTO(
    val id: Long,
    val nombre: String,
    val apellido: String,
    val dni: String,
    val legajo: String,
    val cargo: String,
    val area: String,
    val fechaIngreso: LocalDate
    // salario y liquidaciones: solo si el usuario tiene el rol correcto
)

// Para recibir datos del cliente al crear un empleado
data class EmpleadoNuevoDTO(
    val nombre: String,
    val apellido: String,
    val dni: String,
    val cargoId: Long,
    val areaId: Long
)
```

### Dónde vive el DTO

En la **capa de presentación**, cerca del Controller. El Service no sabe que el DTO existe — trabaja siempre con objetos de dominio.

```
Cliente → DTO de request → Controller → objeto de dominio → Service → BD
BD → objeto de dominio → Controller → DTO de response → Cliente
```

### DTOs de entrada — validación con Bean Validation

Cuando el DTO llega del cliente (body del request), podés declarar validaciones directamente en el DTO con anotaciones de Bean Validation:

```kotlin
data class EmpleadoNuevoDTO(
    @field:NotBlank(message = "El nombre es obligatorio")
    val nombre: String,

    @field:NotBlank
    val apellido: String,

    @field:Size(min = 7, max = 8)
    val dni: String,

    @field:Positive
    val cargoId: Long
)

// En el Controller, @Valid activa las validaciones
@PostMapping("/empleados")
fun crear(@RequestBody @Valid empleadoDTO: EmpleadoNuevoDTO): ResponseEntity<EmpleadoDetalleDTO> { ... }
```

Si falla una validación, Spring devuelve automáticamente un 400 Bad Request con los mensajes de error.

---

## 3. El Mapper — encapsular la traducción

### El problema sin Mapper

Sin una clase dedicada, la conversión dominio → DTO vive en el Controller:

```kotlin
// ❌ MAL: el Controller sabe cómo construir el DTO
@GetMapping("/empleados/{id}")
fun getEmpleado(@PathVariable id: Long): EmpleadoDetalleDTO {
    val empleado = empleadoService.getEmpleado(id)
    return EmpleadoDetalleDTO(    // esta lógica no le pertenece al Controller
        id = empleado.id!!,
        nombre = empleado.nombre,
        apellido = empleado.apellido,
        cargo = empleado.cargo.descripcion,
        area = empleado.area.nombre,
        fechaIngreso = empleado.fechaIngreso
    )
}
```

Si el DTO cambia (se agrega un campo, se renombra otro), modificás el Controller. Si hay 10 endpoints que devuelven `EmpleadoDetalleDTO`, cambiás en 10 lugares.

### La solución: el Mapper

```kotlin
// ✅ BIEN: el Mapper tiene una sola responsabilidad — traducir
@Component
class EmpleadoMapper {

    fun toDetalleDTO(empleado: Empleado): EmpleadoDetalleDTO =
        EmpleadoDetalleDTO(
            id = empleado.id!!,
            nombre = empleado.nombre,
            apellido = empleado.apellido,
            cargo = empleado.cargo.descripcion,
            area = empleado.area.nombre,
            fechaIngreso = empleado.fechaIngreso
        )

    fun toResumenDTO(empleado: Empleado): EmpleadoResumenDTO =
        EmpleadoResumenDTO(
            id = empleado.id!!,
            nombre = empleado.nombre,
            cargo = empleado.cargo.descripcion,
            area = empleado.area.nombre
        )

    fun toDomain(dto: EmpleadoNuevoDTO, cargo: Cargo, area: Area): Empleado =
        Empleado(
            nombre = dto.nombre,
            apellido = dto.apellido,
            dni = dto.dni,
            cargo = cargo,
            area = area
        )
}
```

El Controller ahora delega:

```kotlin
@RestController
@RequestMapping("/empleados")
class EmpleadoController(
    private val empleadoService: EmpleadoService,
    private val empleadoMapper: EmpleadoMapper
) {
    @GetMapping("/{id}")
    fun getEmpleado(@PathVariable id: Long): EmpleadoDetalleDTO {
        val empleado = empleadoService.getEmpleado(id)
        return empleadoMapper.toDetalleDTO(empleado)   // ← el Controller solo orquesta
    }

    @GetMapping
    fun listar(): List<EmpleadoResumenDTO> =
        empleadoService.listarTodos().map { empleadoMapper.toResumenDTO(it) }
}
```

### Por qué importa — tres razones

**Separación de responsabilidades:** el Controller orquesta, el Mapper traduce, el Service opera sobre dominio puro. Cada clase tiene una única razón para cambiar.

**Mantenibilidad:** si el DTO cambia, modificás el Mapper en un único lugar. Los 10 Controllers que lo usan no se enteran.

**Testeabilidad:** podés testear la transformación de forma aislada, sin levantar el contexto de Spring ni mockear nada:

```kotlin
class EmpleadoMapperTest {
    private val mapper = EmpleadoMapper()

    @Test
    fun `convierte un empleado a DTO de resumen correctamente`() {
        val empleado = Empleado(nombre = "Ana", apellido = "García", ...)
        val dto = mapper.toResumenDTO(empleado)
        assertEquals("Ana", dto.nombre)
    }
}
```

### Alternativa: MapStruct

Para proyectos con muchos DTOs, existe [MapStruct](https://mapstruct.org/) — un procesador de anotaciones que genera el código del Mapper automáticamente en tiempo de compilación:

```kotlin
@Mapper(componentModel = "spring")
interface EmpleadoMapper {
    fun toResumenDTO(empleado: Empleado): EmpleadoResumenDTO
    fun toDomain(dto: EmpleadoNuevoDTO): Empleado
}
```

MapStruct infiere el mapeo campo a campo por nombre. Para campos con nombres distintos se agrega `@Mapping(source = "...", target = "...")`. No tiene overhead en runtime — todo se genera en compilación.

---

## 4. El Service — lógica de aplicación

### Responsabilidades

El Service es el corazón de la aplicación. Coordina los casos de uso: qué datos traer, qué validaciones aplicar, qué operaciones ejecutar en qué orden.

```kotlin
@Service
class EmpleadoService(
    private val empleadoRepository: EmpleadoRepository,
    private val cargoRepository: CargoRepository
) {
    @Transactional(readOnly = true)
    fun getEmpleado(id: Long): Empleado =
        empleadoRepository.findById(id)
            .orElseThrow { ResponseStatusException(HttpStatus.NOT_FOUND, "Empleado $id no existe") }

    @Transactional
    fun transferir(empleadoId: Long, nuevaAreaId: Long): Empleado {
        val empleado = getEmpleado(empleadoId)
        val nuevaArea = areaRepository.findById(nuevaAreaId)
            .orElseThrow { ResponseStatusException(HttpStatus.NOT_FOUND, "Área $nuevaAreaId no existe") }
        empleado.transferirA(nuevaArea)    // la lógica de negocio vive en el dominio
        return empleadoRepository.save(empleado)
    }
}
```

### Lo que el Service NO debe hacer

- **No** sabe que existe un DTO. No construye ni parsea JSON.
- **No** maneja el status HTTP. Eso es responsabilidad del Controller.
- **No** contiene lógica de negocio que debería estar en el dominio. `empleado.transferirA(area)` en lugar de `empleado.area = area`.
- **No** llama a otros Services del mismo nivel (ver sección 5).

### Thin controllers, fat domain, lean services

La regla de oro de la arquitectura en capas:

- **El dominio** tiene la lógica de negocio (`empleado.puedeAscender()`, `pedido.calcularTotal()`).
- **El Service** orquesta — busca los objetos necesarios, los conecta, persiste el resultado.
- **El Controller** traduce — convierte request → dominio → response.

Un Service con 200 líneas de lógica condicional es una señal de que esa lógica debería estar en el dominio.

---

## 5. Dependencias entre Services — el error más común

### El problema: dependencia circular

```kotlin
// ❌ MAL: ServiceA llama a ServiceB, ServiceB llama a ServiceA
@Service
class PedidoService(private val clienteService: ClienteService) {
    fun crearPedido(clienteId: Long) {
        val cliente = clienteService.getCliente(clienteId)  // llama a ClienteService
        // ...
    }
}

@Service
class ClienteService(private val pedidoService: PedidoService) {
    fun getHistorial(clienteId: Long) {
        val pedidos = pedidoService.getPedidosDeCliente(clienteId)  // llama a PedidoService
        // ...
    }
}
```

Cuando Spring intenta instanciar estos beans, entra en un loop: para crear `PedidoService` necesita `ClienteService`, para crear `ClienteService` necesita `PedidoService`. En el mejor caso rompe en startup con `BeanCurrentlyInCreationException`. En el peor, genera problemas de performance silenciosos.

### La regla: los llamados siempre van hacia abajo

```
Controller ──→ ServiceA
Controller ──→ ServiceB
ServiceA   ──→ RepoA
ServiceA   ──→ RepoB    ← ✅ un Service puede ir directo a cualquier Repository
ServiceB   ──→ RepoB
```

Si en el `PedidoService` necesitás un dato que "vive" en `ClienteService`, la solución no es llamar a `ClienteService` — es ir directo a `ClienteRepository`:

```kotlin
// ✅ BIEN: PedidoService va directo al repositorio que necesita
@Service
class PedidoService(
    private val pedidoRepository: PedidoRepository,
    private val clienteRepository: ClienteRepository   // ← Repository, no Service
) {
    fun crearPedido(clienteId: Long) {
        val cliente = clienteRepository.findById(clienteId)
            .orElseThrow { ... }
        // ...
    }
}
```

### La otra salida: subir al Controller

El Controller sí puede llamar a múltiples Services sin generar dependencia circular, porque ningún Service lo instancia a él:

```kotlin
// ✅ BIEN: el Controller orquesta múltiples Services
@RestController
class ReporteController(
    private val pedidoService: PedidoService,
    private val clienteService: ClienteService
) {
    @GetMapping("/reportes/cliente/{id}")
    fun reporteCliente(@PathVariable id: Long): ReporteDTO {
        val cliente = clienteService.getCliente(id)
        val pedidos = pedidoService.getPedidosDeCliente(id)
        return reporteMapper.toDTO(cliente, pedidos)
    }
}
```

### Regla general

```
Controller → Service → Repository   ✅
Controller → ServiceA y ServiceB    ✅
ServiceA   → RepoA y RepoB          ✅
ServiceA   → ServiceB               ⚠️  (solo si no hay dependencia circular y está justificado)
ServiceA   → ServiceB → ServiceA    ❌  (circular, nunca)
```

---

## 6. El Controller — responsabilidades concretas

El Controller es el punto de entrada HTTP. Sus responsabilidades son:

- Recibir y parsear el request (path variables, query params, body).
- Validar el formato de entrada (`@Valid`).
- Llamar al Service con los parámetros correctos.
- Convertir el resultado a DTO con el Mapper.
- Responder con el status HTTP correcto.

```kotlin
@RestController
@RequestMapping("/candidatos")
class CandidatoController(
    private val candidatoService: CandidatoService,
    private val candidatoMapper: CandidatoMapper
) {
    // GET /candidatos → 200 OK con lista
    @GetMapping
    fun listar(): List<CandidatoResumenDTO> =
        candidatoService.listarTodos().map { candidatoMapper.toResumenDTO(it) }

    // GET /candidatos/42 → 200 OK o 404 Not Found
    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long): CandidatoDetalleDTO {
        val candidato = candidatoService.getById(id)
        return candidatoMapper.toDetalleDTO(candidato)
    }

    // POST /candidatos → 201 Created con el recurso creado
    @PostMapping
    fun crear(@RequestBody @Valid dto: CandidatoNuevoDTO): ResponseEntity<CandidatoDetalleDTO> {
        val candidato = candidatoService.crear(dto)
        return ResponseEntity.status(HttpStatus.CREATED).body(candidatoMapper.toDetalleDTO(candidato))
    }

    // PUT /candidatos/42 → 200 OK con el recurso actualizado
    @PutMapping("/{id}")
    fun actualizar(@PathVariable id: Long, @RequestBody @Valid dto: CandidatoNuevoDTO): CandidatoDetalleDTO {
        val candidato = candidatoService.actualizar(id, dto)
        return candidatoMapper.toDetalleDTO(candidato)
    }

    // DELETE /candidatos/42 → 204 No Content
    @DeleteMapping("/{id}")
    fun eliminar(@PathVariable id: Long): ResponseEntity<Void> {
        candidatoService.eliminar(id)
        return ResponseEntity.noContent().build()
    }
}
```

### Status HTTP correctos

| Situación | Status |
|---|---|
| Lectura exitosa | 200 OK |
| Creación exitosa | 201 Created |
| Actualización sin body | 204 No Content |
| Recurso no encontrado | 404 Not Found |
| Error de validación | 400 Bad Request |
| Error no manejado | 500 Internal Server Error |

### Manejo de errores con `@ExceptionHandler`

En lugar de manejar excepciones en cada Controller, se puede centralizar con un `@ControllerAdvice`:

```kotlin
@ControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(ResponseStatusException::class)
    fun handleNotFound(ex: ResponseStatusException): ResponseEntity<Map<String, String>> =
        ResponseEntity.status(ex.statusCode).body(mapOf("error" to (ex.reason ?: "Error")))

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<Map<String, List<String?>>> {
        val errores = ex.bindingResult.fieldErrors.map { it.defaultMessage }
        return ResponseEntity.badRequest().body(mapOf("errores" to errores))
    }
}
```

---

## 7. El Repository — acceso a datos

El Repository es el único punto de contacto con la base de datos. No tiene lógica de negocio — solo persiste, consulta y delega en Hibernate.

```kotlin
interface CandidatoRepository : CrudRepository<Candidato, Long> {
    // Spring Data genera el SQL automáticamente por convención de nombres
    fun findByNombre(nombre: String): Optional<Candidato>
    fun findByPartidoId(partidoId: Long): List<Candidato>
    fun findByVotosGreaterThan(votos: Int): List<Candidato>

    // Para queries complejas, JPQL explícito
    @Query("SELECT c FROM Candidato c WHERE c.partido.nombre = :partido AND c.votos > :minVotos")
    fun findDestacadosPor(@Param("partido") partido: String, @Param("minVotos") minVotos: Int): List<Candidato>

    // Entity graph para evitar N+1 en casos específicos
    @EntityGraph(attributePaths = ["partido", "promesas"])
    override fun findById(id: Long): Optional<Candidato>
}
```

---

## 8. Resumen: quién hace qué

| Responsabilidad | Controller | Mapper | Service | Repository | Dominio |
|---|---|---|---|---|---|
| Recibir request HTTP | ✅ | | | | |
| Validar formato de entrada | ✅ | | | | |
| Convertir DTO ↔ dominio | | ✅ | | | |
| Lógica de aplicación | | | ✅ | | |
| Lógica de negocio | | | | | ✅ |
| Demarcación de transacciones | | | ✅ | | |
| Consultar / persistir en BD | | | | ✅ | |
| Responder con status HTTP | ✅ | | | | |
| Saber que existe el DTO | ✅ | ✅ | ❌ | ❌ | ❌ |
| Saber que existe la BD | ❌ | ❌ | ❌ | ✅ | ❌ |

---

## 9. Checklist antes de commitear

- [ ] El Controller no tiene lógica de negocio — delega al Service.
- [ ] El Controller usa el Mapper para construir DTOs, no lo hace a mano.
- [ ] El Service no construye ni parsea DTOs.
- [ ] El Service no llama a otros Services al mismo nivel (o si lo hace, verificar que no haya ciclo).
- [ ] Los DTOs de request tienen validaciones con Bean Validation.
- [ ] Los errores de negocio se lanzan como `ResponseStatusException` con el status correcto.
- [ ] Las lecturas en el Service están marcadas con `@Transactional(readOnly = true)`.
- [ ] Las escrituras en el Service están marcadas con `@Transactional`.
- [ ] El Repository no tiene lógica de negocio — solo queries.
- [ ] La lógica de negocio vive en el dominio, no en el Service.