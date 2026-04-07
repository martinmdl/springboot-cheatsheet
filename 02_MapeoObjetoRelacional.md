# Clase 2 - Mapeo Objeto/Relacional (ORM)

> Material de referencia:
> - [Apunte: Mapeo Objetos Relacional](https://docs.google.com/document/d/1YLmp9vMnSzKg2emt3Bx564Tf1CLalShPc98Z8nCoi7s/edit)
> - [Diapositivas de clase](https://docs.google.com/presentation/d/19O0pK2VvkSCxsekyJFsCSqE2EvMKcKuYHzF_VCVCbYU/edit)
> - [Ejemplo Active Record - Préstamo de Libros en Rails](https://github.com/uqbar-project/eg-prestamo-libros-rails)
> - Videos de clase: [Parte 1](https://drive.google.com/file/d/1UUE84FRrvscSUDDFJ7J0hInQ4beRs4gB/view) | [Parte 2](https://drive.google.com/file/d/1cvXo7AASdJ8bASIR3m_vvmhYHfhbn2ON/view)

---

## 1. El problema: dos mundos distintos

El modelo de objetos y el modelo relacional son estructuralmente incompatibles. Los objetos en memoria forman un **grafo dirigido**: cada objeto es un nodo, cada referencia es una arista. El modelo relacional organiza la información como **tablas de filas y columnas**, sin referencias directas entre registros — solo claves foráneas.

Esta brecha se llama **impedance mismatch** y es el problema central que resuelven los frameworks ORM (Object-Relational Mapper) como JPA/Hibernate.

### Diferencias concretas entre ambos mundos

**Encapsulamiento y comportamiento:** un objeto puede tener un método `activo()` que calcula su estado a partir de otros atributos. En una tabla, el campo `ACTIVO` es un valor crudo (¿`"S"`/`"N"`? ¿`1`/`0"`? ¿`"Y"`/`"N"`?) que requiere documentación externa para interpretarse. Las tablas tienen constraints y triggers, pero no comportamiento propiamente dicho.

**Polimorfismo:** en objetos podés enviarle un mensaje a un objeto sin saber de qué clase concreta es. En SQL, para consultar necesitás saber exactamente qué tablas y columnas están involucradas.

**Herencia:** es un concepto nativo en objetos. En el modelo relacional no existe — hay que elegir entre varias estrategias de mapeo (ver sección 5).

**Variables de clase (`static`/`companion`):** no se pueden asociar a un registro. Si necesitás guardar parámetros globales de la aplicación (porcentaje de descuento VIP, días de mora permitidos), se modela como una tabla de configuración con un único registro.

**Identidad:** en objetos, cada instancia tiene identidad propia garantizada por la VM. Al persistir y recuperar objetos en distintas sesiones, esa identidad se pierde — dos referencias pueden apuntar "al mismo cliente" sin ser el mismo objeto. La BD necesita una clave para garantizar unicidad.

---

## 2. Identidad: clave natural vs. clave subrogada

Al persistir una entidad, necesitamos un campo que identifique unívocamente cada fila. Hay dos enfoques:

**Clave natural:** un subconjunto de campos del negocio (CUIT, número de teléfono, DNI). Ventaja: más legible, simplifica algunas consultas. Desventaja: puede cambiar (el código postal argentino incorporó caracteres extra; en 1998 se agregó el prefijo `4` a los teléfonos), y si participa en relaciones, una modificación propaga cambios a todas las FK.

**Clave subrogada:** un identificador ficticio autogenerado (secuencia, autoincremental, UUID). Es el enfoque que prefieren los frameworks ORM — Hibernate en particular no se lleva bien con claves compuestas.

En el modelo de objetos, el atributo `id` cumple dos roles:
1. Identificar la fila en la tabla.
2. Determinar si el objeto es nuevo (`id == null` → INSERT) o ya existe en la BD (`id != null` → UPDATE).

---

## 3. Conversiones de tipo

El mapeo entre tipos de datos de objetos y tipos de columnas SQL no es trivial:

| Tipo en objetos | Problema en SQL |
|---|---|
| `String` (sin límite) | `VARCHAR` requiere longitud máxima |
| `Int`, `Long`, `Double` | Precisión de decimales, rango; hay que ajustar |
| `LocalDate`, `LocalDateTime` | Múltiples tipos en SQL: `DATE`, `DATETIME`, `TIMESTAMP` |
| `Boolean` | No siempre hay tipo booleano nativo; se guarda como `CHAR(1)`, `INT`, etc. |
| `enum` | Hay que decidir si guardar el nombre, el ordinal, o mapear a una tabla |

Hibernate gestiona estas conversiones automáticamente para los tipos comunes, pero hay que prestarle atención a los enums y fechas.

---

## 4. Cardinalidad de relaciones

### 4.1 Uno a muchos (One-to-Many): Pedido → Ítems

Un pedido tiene muchos ítems; cada ítem pertenece a un solo pedido. En el modelo relacional, **la FK se pone del lado "muchos"** (en la tabla Ítems):

**Opción a — FK no forma parte de la PK del hijo (non-identifying):**
El ítem tiene su propio id autoincremental.

```
ITEMS: ITEM_ID (PK) | PEDIDO_ID (FK) | CANTIDAD
```

**Opción b — FK forma parte de la PK del hijo (identifying):**
El ítem se identifica por la combinación Pedido + Producto. Solo válido si un pedido no puede tener dos ítems del mismo producto.

```
ITEMS: PEDIDO_ID (PK/FK) | PRODUCTO_ID (PK/FK) | CANTIDAD
```

Los frameworks ORM prefieren la opción a (claves subrogadas autoincrementales) porque evitan la complejidad de claves compuestas.

### Tipos de colección y su impacto en el modelo relacional

Al definir una colección en objetos hay que responder: ¿importa el orden? ¿se admiten duplicados?

| Tipo de colección | ¿Columna de orden? | ¿Orden forma parte de la PK? |
|---|---|---|
| `Set` | No | — |
| Conjunto ordenado | Sí | No |
| `List` | Sí | Sí (o autoincremental) |
| `Bag` (admite duplicados, sin orden) | Necesita columna extra para evitar PK duplicada | Sí (o autoincremental) |

Un `Set` es la colección más natural en el modelo relacional (no tiene orden ni duplicados, igual que una relación matemática). Para listas ordenadas hay que agregar una columna `ORDEN` en la tabla hija.

### 4.2 Muchos a uno (Many-to-One): Ítem → Producto

Se resuelve igual que One-to-Many: la FK va del lado "muchos". En la tabla Ítems se agrega `PRODUCTO_ID` apuntando a Productos.

La distinción importa en objetos:
- **Many-to-One:** el objeto A tiene una **referencia** al objeto B.
- **One-to-Many:** el objeto A tiene una **colección** de objetos B.

### 4.3 Muchos a muchos (Many-to-Many): Proveedores y Productos

En el modelo relacional, una relación muchos-a-muchos **siempre requiere una tabla intermedia** (tabla de relación o join table):

```
PROVEEDORES ←→ PROVEEDOR_PRODUCTO ←→ PRODUCTOS
```

**Caso sin atributos propios:** si la relación solo registra la existencia del vínculo (un proveedor vende un producto), alcanza con la tabla intermedia que tenga solo las dos FK.

**Caso con atributos propios:** si la relación tiene datos propios (e.g., cantidad en stock de un producto en un depósito), en objetos ya no alcanza con dos colecciones — se necesita un objeto que modele la relación:

```kotlin
class Stock(val deposito: Deposito, val producto: Producto, val cantidad: Int)
```

En el modelo relacional, la tabla de relación agrega esas columnas extra. La ventaja: desde SQL podés navegar en cualquier dirección sin cambiar el esquema. En objetos tenés que decidir explícitamente la dirección de navegación (y si querés bidireccionalidad, mantener la consistencia de ambas colecciones manualmente).

---

## 5. Herencia en el modelo relacional

El modelo relacional no tiene herencia. Cuando en objetos tenemos una jerarquía (e.g., `Producto` → `ProductoComprado`, `ProductoFabricado`, `ProductoConservado`), hay tres estrategias de mapeo:

### 5.1 Single Table (`SINGLE_TABLE`)

Toda la jerarquía en **una sola tabla**. Se agrega una columna discriminante (`TIPO_PROD`) que indica la clase concreta de cada fila. Los atributos exclusivos de cada subclase conviven en la misma tabla — los que no aplican quedan en `null`.

```
PRODUCTOS: ID | TIPO_PROD | NOMBRE | VALOR | PESO | DIAS | PRECIO | CANT_HORAS
```

**A favor:** consultas simples (una sola tabla, sin JOINs), buen rendimiento, soporta relaciones polimórficas sin complicaciones.

**En contra:** columnas nulas para atributos que no aplican a todas las subclases; riesgo de "reutilizar" campos para cosas distintas; el campo discriminante es obligatorio.

**Conviene cuando:** las subclases comparten muchos atributos entre sí y querés evitar JOINs.

### 5.2 Table Per Class (`TABLE_PER_CLASS`)

**Una tabla por cada clase concreta**, sin tabla para la superclase. Cada tabla repite los atributos heredados.

```
COMPRADOS:  ID | NOMBRE | VALOR | PESO | PRECIO
FABRICADOS: ID | NOMBRE | VALOR | PESO | CANT_HORAS
CONSERVADOS: ID | NOMBRE | VALOR | PESO | DIAS | PRECIO
```

**A favor:** permite campos `NOT NULL` por subclase; no necesita discriminante.

**En contra:** para consultas polimórficas (¿qué productos pesan más de 1 kg?) hay que hacer `UNION` de todas las tablas; **no soporta FK cuando la relación es `*-to-one`** (el proveedor que referencia a "un producto cualquiera" no puede tener FK contra tres tablas distintas); los atributos heredados se repiten.

**Conviene cuando:** las subclases comparten muy pocos atributos entre sí, o son entidades conceptualmente independientes.

### 5.3 Joined (`JOINED`)

**Una tabla por cada clase** (tanto abstractas como concretas). La tabla de la superclase tiene los atributos comunes; las subclases tienen solo sus atributos propios y su PK es también FK a la tabla madre.

```
PRODUCTOS:   ID | NOMBRE | VALOR | PESO
COMPRADOS:   ID (PK/FK) | PRECIO
FABRICADOS:  ID (PK/FK) | CANT_HORAS
CONSERVADOS: ID (PK/FK) | DIAS | PRECIO
```

**A favor:** modelo más cercano a la normalización (sin atributos repetidos), soporta relaciones polimórficas naturalmente, permite `NOT NULL` en atributos propios.

**En contra:** la opción más costosa en performance — recuperar un objeto requiere JOINs adicionales; guardar un objeto requiere múltiples INSERTs en la misma transacción.

**Conviene cuando:** hay muchos atributos comunes pero también varios propios en cada subclase, y el volumen de datos no es crítico.

### Tabla comparativa

| Estrategia | Consultas polimórficas | Relaciones `*-to-one` | Columnas nulas | JOINs para recuperar |
|---|---|---|---|---|
| `SINGLE_TABLE` | Simple (una tabla) | Soportadas | Sí | Ninguno |
| `TABLE_PER_CLASS` | UNION de N tablas | **No soportadas con FK** | No | Ninguno |
| `JOINED` | LEFT JOIN de N tablas | Soportadas | No | N+1 JOINs |

---

## 6. Hidratación: qué traer y cuándo

**Hidratar** un objeto es poblarlo con datos desde la base. El problema es decidir cuánto del grafo traer en cada consulta.

### El dilema: ¿traer poco o traer todo?

Para una pantalla que solo muestra fecha y total del pedido, hacer 4 JOINs (pedido → cliente → ítems → productos) es un costo innecesario. Para una pantalla que muestra el detalle completo del pedido, esos JOINs son necesarios.

La decisión es por **caso de uso**: no existe una estrategia única correcta.

### Lazy loading

La asociación *lazy* (carga perezosa) es la solución del ORM para esta tensión. La colección de ítems del pedido no se carga al traer el pedido — en su lugar, Hibernate pone un **proxy** (ver Clase 3). El proxy dispara la consulta a la BD **solo cuando algo accede a la colección**:

```kotlin
val pedido = pedidoRepository.findById(1)
// Hasta acá: solo se consultó la tabla PEDIDOS

val masoCaro = pedido.items.maxOf { it.precioTotal() }
// Acá: el proxy dispara SELECT * FROM ITEMS WHERE PEDIDO_ID = 1
```

**Implicancia práctica:** si la sesión de Hibernate ya cerró cuando intentás acceder a la colección, obtenés `LazyInitializationException`. Por eso `open-in-view` y el diseño de las transacciones importan tanto (ver Clase 3).

### Ciclo de vida y cascada

Las entidades pueden tener ciclos de vida dependientes o independientes:

**Ciclo de vida compartido (cascada):** los ítems no tienen sentido sin su pedido. Si se elimina un pedido, sus ítems deben eliminarse. Si se crea un pedido, sus ítems se persisten juntos. Esto se configura con `CascadeType.ALL` + `orphanRemoval = true`.

**Ciclo de vida independiente:** los productos existen independientemente de los pedidos — se actualizan por separado, en su propio caso de uso. No tiene sentido en cascada aquí.

> **Para el TP:** identificar los ciclos de vida antes de configurar la cascada. Una cascada mal configurada puede eliminar entidades que deberían sobrevivir, o dejar huérfanos que generan inconsistencias.

---

## 7. Arquitecturas de acceso a datos

### Data Mapper / Repository

El acceso a datos se centraliza en un objeto separado del dominio: el **Repository** (también llamado DAO o Home). El objeto de dominio no sabe nada de la BD; el repositorio se encarga de todo:

```kotlin
// El dominio no tiene lógica de persistencia
class Cliente(val nombre: String, val email: String)

// El repositorio encapsula el acceso a datos
interface ClienteRepository : JpaRepository<Cliente, Long> {
    fun findByNombre(nombre: String): List<Cliente>
}
```

Es el patrón que usa JPA/Spring Boot y el que vamos a seguir en la materia. **Thin controllers, fat domain, lean repositories** — el dominio tiene la lógica de negocio, el repositorio solo persiste y consulta.

### Active Record

El objeto de dominio incorpora las responsabilidades de persistencia directamente:

```ruby
# En Rails/Active Record
cliente = Cliente.new(nombre: "Ana")
cliente.save           # INSERT
Cliente.find_by(nombre: "Ana")  # SELECT
cliente.destroy        # DELETE
```

Es el patrón de Rails y otros frameworks. Más cómodo para casos simples, pero mezcla responsabilidades y dificulta el testing unitario (el objeto de dominio depende de la BD).

### Mapeo manual vs. framework ORM

**Mapeo manual (JDBC puro):** escribís los queries SQL a mano, manejás conexiones, transacciones y la conversión de `ResultSet` a objetos.

- Ventaja: control total, performance óptima para queries complejos.
- Desventaja: repetitivo, propenso a errores, bajo nivel (abrir/cerrar conexiones, manejar transacciones explícitamente), difícil de mantener.

**Framework ORM (JPA/Hibernate):** el mapeo es declarativo con anotaciones. El framework genera los queries, gestiona conexiones y transacciones.

- Ventaja: menos código, mayor nivel de abstracción, transaccionalidad declarativa (`@Transactional`).
- Desventaja: hay que entender qué queries genera el framework para evitar sorpresas (como el problema N+1).

---

## 8. ¿Objetos primero o base primero? (O/R vs. R/O)

Una decisión de diseño inicial importante: ¿diseñás primero el modelo de objetos y de ahí derivás las tablas, o diseñás primero las tablas y adaptás el modelo de objetos?

**Base primero (R/O):** tiene sentido cuando la BD ya existe (sistema legado), o cuando múltiples sistemas acceden al mismo origen de datos. La BD es la fuente de verdad.

**Objetos primero (O/R):** el modelo de objetos captura las reglas de negocio; las tablas son un detalle de persistencia. Es el enfoque que asume JPA con `ddl-auto: create-drop` o `update` en desarrollo.

En la práctica no existe pureza en ninguno de los dos enfoques — siempre hay que hacer concesiones en ambos mundos. Lo importante es entender qué decisiones estás tomando y por qué.

---

## 9. Resumen de decisiones de diseño en el mapeo O/R

Al diseñar la persistencia de un sistema, estas son las preguntas que hay que responder:

**Identidad:** ¿clave natural o subrogada? Si subrogada, ¿autoincremental o UUID?

**Cardinalidad:** ¿cuál es la dirección de la relación? ¿dónde va la FK? ¿es unidireccional o bidireccional en objetos?

**Colecciones:** ¿importa el orden? ¿se admiten duplicados? Eso determina el tipo de colección y si necesitás columna de orden.

**Herencia:** ¿Single Table, Table Per Class, o Joined? Depende de cuántos atributos comparten las subclases y cuántos JOINs podés tolerar.

**Hidratación:** ¿qué parte del grafo traer en cada caso de uso? ¿lazy o eager para cada relación?

**Ciclo de vida:** ¿las entidades relacionadas tienen ciclo de vida compartido (cascada) o independiente?

**Arquitectura:** ¿Repository (Data Mapper) o Active Record? ¿ORM o mapeo manual?

**O/R o R/O:** ¿quién manda, el modelo de objetos o el relacional?