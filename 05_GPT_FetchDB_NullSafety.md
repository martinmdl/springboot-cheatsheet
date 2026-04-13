# Kotlin + Spring Data JPA: `getById`, `findById`, proxies y null-safety

## Índice

1. [Introducción](#1-introducción)
2. [`getById` y proxies en JPA](#2-getbyid-y-proxies-en-jpa)
3. [¿Qué es un proxy y cómo funciona internamente?](#3-qué-es-un-proxy-y-cómo-funciona-internamente)
4. [`Optional<T>` vs Proxy](#4-optionalt-vs-proxy)
5. [Null safety en Kotlin (`?`, `?.`, `?:`, `!!`)](#5-null-safety-en-kotlin----)
6. [Métodos de acceso a datos en Spring Data JPA](#6-métodos-de-acceso-en-spring-data-jpa)
7. [Recomendaciones en Kotlin](#7-recomendaciones-en-kotlin)
8. [Ejemplos completos](#8-ejemplos-completos)
9. [Fuentes](#9-fuentes)

---

## 1. Introducción

En aplicaciones Kotlin con Spring Data JPA existen varias formas de obtener entidades desde la base de datos. Las diferencias entre `getById`, `findById` y `findByIdOrNull` no son triviales y afectan:

* Cuándo se ejecuta la query
* Cómo se manejan valores inexistentes
* Qué tipo se devuelve
* Qué excepciones pueden ocurrir

---

## 2. `getById` y proxies en JPA

```kotlin
val book = bookRepository.getById(idBook)
```

Este método **NO devuelve `null`**.

En cambio:

* Devuelve un **proxy**
* No ejecuta una query inmediatamente
* La excepción ocurre más tarde

### Comportamiento

```kotlin
val book = bookRepository.getById(idBook)
println(book.title) // ← aquí puede fallar
```

Si el registro no existe:

* Se lanza `EntityNotFoundException` al acceder a una propiedad

---

## 3. ¿Qué es un proxy y cómo funciona internamente?

Un proxy es un objeto que:

* Representa una entidad real
* Contiene inicialmente solo el `id`
* Está gestionado por el contexto de persistencia (Hibernate Session)

### Flujo interno

1. `getById(id)` → devuelve proxy
2. No hay acceso a DB todavía
3. Acceso a propiedad (`book.title`)
4. Hibernate ejecuta:

   ```sql
   SELECT * FROM book WHERE id = ?
   ```
5. Resultado:

   * Existe → hidrata el objeto
   * No existe → `EntityNotFoundException`

### Casos importantes

* Si la sesión está cerrada:

  * `LazyInitializationException`
* Mejora performance al diferir carga (lazy loading)

---

## 4. `Optional<T>` vs Proxy

`Optional<T>` **NO es un proxy**.

Es un contenedor de valor.

```kotlin
val opt: Optional<Book> = bookRepository.findById(id)
```

### Características

* Representa presencia o ausencia
* No tiene lazy loading
* No ejecuta lógica diferida

### Uso

```kotlin
if (opt.isPresent) {
    val book = opt.get()
}
```

O más idiomático en Java:

```java
opt.orElse(null)
```

---

## 5. Null safety en Kotlin

Kotlin distingue explícitamente entre tipos nullables y no nullables.

### Tipos

```kotlin
var book: Book?   // puede ser null
var book2: Book   // no puede ser null
```

---

### Safe call (`?.`)

```kotlin
book?.title
```

Equivalente a:

```kotlin
if (book != null) book.title else null
```

---

### Elvis operator (`?:`)

```kotlin
val title = book?.title ?: "sin título"
```

---

### Not-null assertion (`!!`)

```kotlin
book!!.title
```

* Si `book == null` → `KotlinNullPointerException`

---

## 6. Métodos de acceso en Spring Data JPA

### `getById(id)`

* Devuelve proxy
* Lazy loading
* Puede lanzar excepción al acceder
* No retorna null

---

### `findById(id)`

```kotlin
val book: Optional<Book> = bookRepository.findById(id)
```

* Estilo Java
* Verboso en Kotlin

---

### `findByIdOrNull(id)` (Kotlin)

```kotlin
val book: Book? = bookRepository.findByIdOrNull(id)
```

* Extensión de Kotlin
* Devuelve nullable
* Más idiomático

---

## 7. Recomendaciones en Kotlin

Uso general:

* Preferir `findByIdOrNull`
* Evitar `getById` salvo que necesites lazy loading explícito
* Evitar `Optional` en Kotlin

Tabla resumen:

| Método         | Tipo devuelto | Lazy | Idiomático Kotlin |
| -------------- | ------------- | ---- | ----------------- |
| getById        | Proxy         | Sí   | No                |
| findById       | Optional<T>   | No   | No                |
| findByIdOrNull | T?            | No   | Sí                |

---

## 8. Ejemplos completos

### Caso recomendado

```kotlin
val book = bookRepository.findByIdOrNull(id)

val title = book?.title ?: "sin título"
```

---

### Caso con `getById`

```kotlin
val book = bookRepository.getById(id)

// Puede fallar aquí si no existe
println(book.title)
```

---

### Caso con Optional

```kotlin
val opt = bookRepository.findById(id)

if (opt.isPresent) {
    println(opt.get().title)
}
```

---

## 9. Fuentes

* Spring Data JPA Reference Documentation
* Hibernate ORM Documentation
* Kotlin Language Documentation (Null Safety)
* Java Optional API (java.util.Optional)