# Clase 1 — Persistencia, Modelo Relacional y Docker

> Material de referencia:
> - [Apunte: Introducción a Persistencia](https://docs.google.com/document/d/1nCy-Xk00lBUrBFQvTWk9P5xsw8ee6JOVklSUlRN3mUI/edit)
> - [Apunte: Modelo Relacional](https://docs.google.com/document/d/1uF3yoYIFmLxTH5ZJoT9I3cc5TW9b-H3BqZJbLudKBcA/edit)
> - [Apunte: Normalización](https://docs.google.com/document/d/1Jil-3oiveXDtY1iKBCof7jE9ooRFJ-f1KjcXgaGk6F0/edit)
> - [Taller de Docker](https://docs.google.com/document/d/16-ZVmZQrCbFDDnEyI8eABSp2rwsw3bz1WYyJ7DM9Rxw/edit)
> - [Docker Compose docs](https://docs.docker.com/compose/)
> - [PostgreSQL docs](https://www.postgresql.org/docs/current/)
> - [Spring Boot + JPA](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.jpa-and-spring-data)

---

## 1. Persistencia — conceptos mínimos

Los objetos en RAM se pierden cuando termina el proceso. Persistir es guardar estado fuera del proceso para recuperarlo luego.

```kotlin
// Estado en memoria (volátil — muere con el proceso)
val pedido = Pedido(total = 1200)

// Estado persistido (durable — sobrevive reinicios)
pedidoRepository.save(pedido)
```

**Preguntas de diseño que siempre hay que responder:**
- ¿Qué guardar y con qué estructura?
- ¿Cuándo guardar y cuándo consultar?
- ¿Cómo mantener la consistencia ante errores?
- ¿Cómo versionar cambios de schema a lo largo del tiempo?

### Estrategias de arquitectura

**A. Ambiente vivo** — los objetos en RAM son la fuente de verdad, la BD es un backup. Riesgo: si algo falla a mitad de una operación, el rollback es complejo porque hay que deshacer cambios en memoria y en la BD.

**B. Ambiente recreado por request** — cada request arranca con un ambiente vacío, hidrata desde la BD lo que necesita, ejecuta la lógica y persiste. Es el modelo que usa Spring Boot con JPA. Ventaja: la BD gestiona la consistencia y el rollback.

> Para la materia y el TP siempre usamos **B**.

---

## 2. Transacciones y ACID

Una **transacción** es un conjunto de operaciones que se tratan como una unidad: o se completan todas, o no se completa ninguna.

| Propiedad | Qué garantiza |
|---|---|
| **Atomicidad** | Todo o nada — si falla una operación, se deshacen todas |
| **Consistencia** | La BD pasa de un estado válido a otro estado válido |
| **Aislamiento** | Las transacciones concurrentes no se interfieren entre sí |
| **Durabilidad** | Un commit confirmado sobrevive a fallas del sistema |

### Niveles de aislamiento

El aislamiento tiene un costo de performance. PostgreSQL permite configurarlo:

| Nivel | Qué evita | Cuándo usar |
|---|---|---|
| `READ_UNCOMMITTED` | Nada | Casi nunca |
| `READ_COMMITTED` | Lecturas sucias | Default de PostgreSQL — suficiente para la mayoría |
| `REPEATABLE_READ` | Lecturas no repetibles | Reportes o procesos batch |
| `SERIALIZABLE` | Máximo rigor | Operaciones críticas de consistencia |

```kotlin
// En Spring, se puede configurar por método
@Transactional(isolation = Isolation.READ_COMMITTED)
fun transferir(origen: Long, destino: Long, monto: Double) { ... }
```

> **Referencia:** [PostgreSQL Isolation Levels](https://www.postgresql.org/docs/current/transaction-iso.html)

---

## 3. Modelo relacional — lo esencial

### Claves

**PK (Primary Key):** identifica una fila de forma única. No puede ser nula. Opciones:
- **Clave natural:** dato del negocio (CUIT, DNI). Legible, pero puede cambiar.
- **Clave subrogada:** id autoincremental o UUID generado por la BD. Estable, preferida por los ORMs.

**FK (Foreign Key):** referencia la PK de otra tabla. Define integridad referencial — no podés tener un ítem que apunte a un pedido que no existe.

### Integridad referencial — qué pasa al borrar el padre

```sql
-- Si borrás un pedido, ¿qué pasa con sus ítems?
FOREIGN KEY (pedido_id) REFERENCES pedido(id)
  ON DELETE RESTRICT    -- no deja borrar el padre si tiene hijos
  ON DELETE CASCADE     -- borra los hijos automáticamente
  ON DELETE SET NULL    -- desvincula los hijos (pedido_id = null)
```

**Cómo elegir:**
- `RESTRICT` cuando los hijos no tienen sentido sin el padre pero querés controlarlo explícitamente.
- `CASCADE` cuando los hijos son parte del mismo ciclo de vida que el padre (ítems de un pedido, promesas de un candidato).
- `SET NULL` cuando el hijo puede existir sin el padre (un empleado puede existir aunque se elimine su área).

### Ejemplo SQL completo

```sql
CREATE TABLE pedido (
  id        BIGSERIAL    PRIMARY KEY,
  cliente   VARCHAR(120) NOT NULL,
  fecha     DATE         NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE item_pedido (
  id         BIGSERIAL    PRIMARY KEY,
  pedido_id  BIGINT       NOT NULL,
  producto   VARCHAR(120) NOT NULL,
  cantidad   INT          NOT NULL CHECK (cantidad > 0),
  precio     NUMERIC(10,2) NOT NULL,
  CONSTRAINT fk_item_pedido
    FOREIGN KEY (pedido_id)
    REFERENCES pedido(id)
    ON DELETE CASCADE
);
```

> **Referencia:** [PostgreSQL Constraints y FK](https://www.postgresql.org/docs/current/ddl-constraints.html)

### Normalización — en una línea

Separar datos en tablas para evitar redundancia. Si la misma información aparece en múltiples filas, probablemente necesitás otra tabla.

```
❌ Antes: pedido(cliente, producto1, producto2, producto3)
✅ Después: pedido(cliente) + item_pedido(pedido_id, producto, cantidad)
```

---

## 4. Docker — qué es y qué problema resuelve

### El problema sin Docker

Tu proyecto necesita PostgreSQL corriendo localmente. Sin Docker:
- Instalás PostgreSQL en tu máquina.
- Tu compañero de TP lo instala en la suya con otra versión.
- En el servidor de CI/CD hay otra versión más.
- "En mi máquina funciona" — el problema clásico.

### Qué hace Docker

Docker empaqueta un servicio (la BD, el cliente web, la app) junto con todo lo que necesita para correr, en una unidad estandarizada llamada **contenedor**. Cualquier máquina con Docker instalado puede correr ese contenedor de forma idéntica.

**Para la materia, Docker te da:**

- **Entorno reproducible:** todos en el equipo tienen la misma versión de PostgreSQL con la misma configuración. No hay diferencias entre máquinas.
- **Sin instalación de PostgreSQL:** no ensuciás tu sistema operativo. El motor vive dentro del contenedor y lo podés destruir cuando quieras.
- **Reseteo fácil:** `docker compose down --volumes` destruye la BD y `docker compose up` la recrea desde cero. Ideal para development.
- **Múltiples proyectos en paralelo:** cada proyecto tiene su propia BD en su propio contenedor, en puertos distintos, sin conflictos.
- **pgAdmin incluido:** el cliente web para explorar la BD también corre en un contenedor, sin instalación adicional.

### Conceptos mínimos que necesitás saber

**Imagen:** plantilla de solo lectura. `postgres:16-alpine` es la imagen oficial de PostgreSQL versión 16 en su variante liviana.

**Contenedor:** instancia en ejecución de una imagen. Podés tener múltiples contenedores del mismo tipo corriendo a la vez.

**Volumen:** carpeta compartida entre el contenedor y tu máquina. Sirve para que los datos de la BD sobrevivan aunque detengas y reinicies el contenedor.

**`docker-compose.yml`:** archivo que define qué contenedores levantar, cómo configurarlos y cómo se conectan entre sí. Un solo comando (`docker compose up`) levanta todo.

---

## 5. Implementar Docker en tu proyecto Spring Boot

### Estructura de archivos resultante

```
tu-proyecto/
├── src/
│   └── main/
│       └── resources/
│           └── application.yml      ← configuración de Spring Boot
├── docker-compose.yml               ← definición de los contenedores
└── build.gradle.kts
```

### Paso 1 — el `docker-compose.yml`

Creá este archivo en la raíz del proyecto:

```yaml
services:
  db:
    image: postgres:16-alpine            # versión de PostgreSQL — elegí una y fijala
    container_name: mi_proyecto_db
    environment:
      POSTGRES_DB: mi_proyecto           # nombre de la base de datos
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"                      # puerto_host:puerto_contenedor
    volumes:
      - postgres_data:/var/lib/postgresql/data   # los datos sobreviven al reinicio

  pgadmin:
    image: dpage/pgadmin4
    container_name: mi_proyecto_pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@demo.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db                              # pgAdmin espera a que la BD esté lista

volumes:
  postgres_data:                        # Docker administra este volumen
```

### Paso 2 — el `application.yml` de Spring Boot

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mi_proyecto
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver

  jpa:
    open-in-view: false                 # siempre false — ver Clase 3
    ddl-auto: create-drop               # solo para development (ver tabla abajo)
    show-sql: true                      # útil para ver los queries generados
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

**Opciones de `ddl-auto` — cuál elegir:**

| Valor | Comportamiento | Cuándo |
|---|---|---|
| `create-drop` | Destruye y recrea el schema en cada inicio | Development activo — iterando el modelo |
| `update` | Intenta ALTER TABLE automáticos | ⚠️ Peligroso: no tiene rollback, puede perder datos |
| `validate` | Verifica que el mapeo coincida con el schema | Producción + Flyway |
| `none` | No hace nada | Producción gestionada externamente |

> **Para el TP en etapa inicial:** `create-drop`. Cuando el modelo se estabilice y empecés a usar Flyway: `validate`.

### Paso 3 — el driver de PostgreSQL en `build.gradle.kts`

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")   // driver JDBC de PostgreSQL
    // ... resto de dependencias
}
```

### Paso 4 — levantar el entorno

```bash
# Levantar todos los contenedores en background
docker compose up -d

# Ver los logs de la BD (útil si algo no levanta)
docker compose logs db

# Ver qué contenedores están corriendo
docker compose ps
```

Una vez levantado, Spring Boot puede conectarse a `localhost:5432` y pgAdmin está en `http://localhost:5050`.

---

## 6. Conectarse a pgAdmin

1. Abrí `http://localhost:5050` en el navegador.
2. Ingresá con `admin@demo.com` / `admin` (los valores del `docker-compose.yml`).
3. Click derecho en **Servers → Register → Server**.
4. En la pestaña **General**: ponele un nombre (ej: `mi_proyecto`).
5. En la pestaña **Connection**:
   - **Host:** `db` ← el nombre del servicio en el `docker-compose.yml`, **no** `localhost`
   - **Port:** `5432`
   - **Username:** `postgres`
   - **Password:** `postgres`

> **Por qué `db` y no `localhost`:** pgAdmin corre dentro de un contenedor. Desde adentro de la red de Docker, `localhost` es el propio contenedor de pgAdmin. Para llegar a otro contenedor, usás el nombre del servicio (`db`). Desde tu máquina host, en cambio, usás `localhost:5432`.

---

## 7. Decisiones a tomar — guía rápida

### ¿Qué versión de PostgreSQL usar?

```yaml
image: postgres:16-alpine   # ✅ versión estable reciente, liviana
image: postgres:latest      # ⚠️ puede romperse si sale una nueva major version
image: postgres:15-alpine   # también válido
```

**Consejo:** elegí una versión específica y fijala (`16-alpine`, `15`, etc.). `latest` puede traer sorpresas cuando actualizan la imagen.

### ¿Con o sin volumen persistente?

```yaml
# Con volumen — los datos sobreviven al `docker compose down`
volumes:
  - postgres_data:/var/lib/postgresql/data

# Sin volumen — la BD se borra al detener el contenedor
# (no pongas nada en volumes)
```

**Para development:** con volumen es más cómodo (no perdés datos al reiniciar). Para resetear completamente, usás `docker compose down --volumes`.

**Para CI/CD o tests:** sin volumen — querés empezar siempre de cero.

### ¿Qué puerto exponer?

```yaml
ports:
  - "5432:5432"   # puerto_host:puerto_contenedor
```

El puerto del contenedor siempre es `5432` (el default de PostgreSQL). El puerto del host lo podés cambiar si ya tenés algo corriendo en `5432`:

```yaml
ports:
  - "5433:5432"   # accedés desde tu máquina en localhost:5433
```

Si hacés esto, actualizá también el `application.yml`:
```yaml
url: jdbc:postgresql://localhost:5433/mi_proyecto
```

### ¿Incluir pgAdmin en el `docker-compose.yml`?

pgAdmin es opcional — es solo un cliente web para explorar la BD visualmente. Alternativas:
- **pgAdmin en Docker:** cómodo, sin instalación, funciona bien para el TP.
- **IntelliJ IDEA:** tiene un cliente de BD integrado (pestaña Database) que funciona muy bien.
- **DBeaver:** cliente desktop gratuito y multiplataforma.

Para el TP, pgAdmin en Docker es la opción más reproducible porque todos tienen exactamente el mismo cliente.

---

## 8. Comandos de referencia

```bash
# Levantar todo
docker compose up -d

# Ver logs en tiempo real
docker compose logs -f db

# Detener sin borrar datos
docker compose down

# Detener Y borrar todos los datos de la BD (reset total)
docker compose down --volumes

# Conectarse a la BD desde la terminal (sin pgAdmin)
docker compose exec db psql -U postgres -d mi_proyecto

# Ver qué contenedores están corriendo
docker compose ps

# Reiniciar un contenedor específico
docker compose restart db
```

---

## 9. Problemas frecuentes y cómo resolverlos

**`Connection refused` al iniciar Spring Boot:**
- Verificá que los contenedores estén corriendo: `docker compose ps`.
- Verificá que el puerto en `application.yml` coincida con el expuesto en `docker-compose.yml`.
- Esperá unos segundos — PostgreSQL tarda en inicializarse. Podés ver cuándo está listo con `docker compose logs db`.

**pgAdmin no puede conectarse a la BD:**
- El host debe ser el nombre del servicio (`db`), no `localhost`.
- Verificá que ambos contenedores estén en la misma red (por defecto, todos los servicios del mismo `docker-compose.yml` lo están).

**`Password authentication failed`:**
- Las credenciales en `application.yml` deben coincidir exactamente con `POSTGRES_USER` y `POSTGRES_PASSWORD` en el `docker-compose.yml`.

**`Table doesn't exist` al levantar Spring Boot:**
- Con `ddl-auto: create-drop` Hibernate crea las tablas. Si ves este error, probablemente tenés `ddl-auto: validate` o `none` sin haber creado las tablas manualmente.

**El puerto 5432 ya está en uso:**
- Tenés PostgreSQL instalado localmente corriendo en ese puerto. Cambiá el puerto del host: `"5433:5432"` y actualizá el `application.yml`.

---

## 10. Checklist antes de hacer `docker compose up`

- [ ] `docker-compose.yml` en la raíz del proyecto con versión fija de PostgreSQL.
- [ ] Credenciales iguales en `docker-compose.yml` y `application.yml`.
- [ ] `runtimeOnly("org.postgresql:postgresql")` en `build.gradle.kts`.
- [ ] `ddl-auto: create-drop` para development, `validate` para producción.
- [ ] `open-in-view: false` en `application.yml`.
- [ ] Puerto del host libre (o cambiado si hay conflicto).
- [ ] Volumen definido si querés persistir datos entre reinicios.