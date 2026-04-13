# Proxies JPA, Optional y Null Safety en Kotlin

> Referencias: [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories) | [Hibernate Proxy](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#fetching-strategies) | [Kotlin Null Safety](https://kotlinlang.org/docs/null-safety.html)

---

## ¿Qué devuelve `getById` si el ID no existe?

No devuelve `null` ni lanza excepción inmediatamente. Devuelve un **proxy** — la excepción aparece al acceder a sus propiedades:

```kotlin
val book = bookRepository.getById(999L)  // ok — devuelve proxy
println(book.title)                       // aquí: EntityNotFoundException
```

**¿Cuándo usar `getById`?** Solo cuando necesitás una referencia para construir una FK sin fetchear el objeto completo:

```kotlin
val author = authorRepository.getById(authorId)  // proxy — no va a la BD
bookRepository.save(Book(title = "...", author = author))  // Hibernate usa el id del proxy
```

---

## ¿Qué es exactamente un proxy?

Un objeto sustituto generado por Hibernate en runtime (via ByteBuddy). Solo contiene el `id`. Al acceder a cualquier otra propiedad, dispara el `SELECT`:

- **Registro existe** → hidrata el objeto.
- **Registro no existe** → `EntityNotFoundException`.
- **Sesión cerrada** → `LazyInitializationException` (típico con `open-in-view: false`).

---

## ¿`Optional<T>` es lo mismo que un proxy?

No. `Optional` es un contenedor de valor sin comportamiento lazy — el `SELECT` ya se ejecutó cuando lo tenés en mano.

```kotlin
val opt = bookRepository.findById(id)   // SELECT ejecutado
opt.isPresent                           // true/false, sin más queries
```

Es la API de Java. En Kotlin hay algo mejor.

---

## ¿Cuál es el método estándar en Kotlin?

`findByIdOrNull` — extensión de Spring Data que devuelve `Book?` en lugar de `Optional<Book>`:

```kotlin
val book: Book? = bookRepository.findByIdOrNull(id)
```

Internamente es `findById(id).orElse(null)`. Más idiomático que `Optional` en Kotlin.

**Comparativa rápida:**

| Método | Devuelve | Va a la BD | Usar cuando |
|---|---|---|---|
| `getById` | Proxy | Al acceder propiedades | Solo para construir asociaciones |
| `findById` | `Optional<Book>` | Inmediatamente | Interop con Java |
| `findByIdOrNull` ✅ | `Book?` | Inmediatamente | Default en Kotlin |

---

## ¿Cómo funciona el `?` en Kotlin?

Es parte del **tipo**, no un operador. Le dice al compilador que la variable puede ser `null`:

```kotlin
var book: Book    // nunca null — garantía en compilación
var book: Book?   // puede ser Book o null
```

**Operadores para trabajar con nullable:**

```kotlin
book?.title               // safe call: devuelve null si book es null
book?.title ?: "default"  // Elvis: valor por defecto si es null
book!!.title              // not-null assertion: lanza KotlinNullPointerException si es null (evitar)
```

---

## Patrón estándar en el Service

```kotlin
@Transactional(readOnly = true)
fun getBook(id: Long): Book =
    bookRepository.findByIdOrNull(id)
        ?: throw ResponseStatusException(HttpStatus.NOT_FOUND, "Book $id no existe")
```

El `?:` convierte el `Book?` en `Book` — si es null lanza el 404, si no el compilador sabe que es non-nullable y no te pide más verificaciones.