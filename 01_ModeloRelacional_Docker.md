# Clase 1 — Persistencia, Modelo Relacional y Docker (guía práctica)

> Versión resumida y orientada a decisiones de diseño.

---

## 1. Objetivos de la clase

Al terminar deberías poder:
- Elegir un medio de persistencia según el problema.
- Entender por qué existe el choque objeto-relacional.
- Diseñar tablas con PK/FK e integridad.
- Levantar PostgreSQL + pgAdmin con Docker para practicar.

---

## 2. Persistencia en 2 minutos

### Idea base
Los objetos en RAM se pierden cuando termina el proceso. Persistir = guardar estado fuera del proceso para recuperarlo luego.

### Preguntas de diseño (siempre)
- Que guardar.
- Cuando guardar.
- Como versionar cambios de schema.
- Donde ejecutar la logica critica (service o base).

### Mini ejemplo
```kotlin
// Estado en memoria (volatil)
val pedido = Pedido(total = 1200)

// Estado persistido (durable)
pedidoRepository.save(pedido)
```

---

## 3. Medios persistentes: comparativa rapida

| Medio | Ventaja | Riesgo/limite | Cuando usar |
|---|---|---|---|
| Snapshot de memoria | Muy simple | Guarda todo, alto costo | Casos chicos/monousuario |
| Archivos (JSON/XML/binario) | Portable, controlable | Manejo manual de concurrencia y consistencia | Apps simples o export/import |
| Base relacional (PostgreSQL) | Transacciones, integridad, concurrencia | Requiere modelado y SQL | Backends multiusuario |

---

## 4. Estrategias de arquitectura de persistencia

## A. Ambiente vivo (objetos como fuente principal)
- Logica sobre objetos en RAM.
- Se sincroniza con BD.
- Riesgo: rollback complejo si falla en mitad de operacion.

## B. Ambiente recreado por request (Spring Boot clasico)
- Cada request hidrata desde BD lo necesario.
- Ejecuta logica.
- Persiste y cierra transaccion.
- Ventaja: consistencia y rollback mas predecibles.

Regla practica:
- Si priorizas simplicidad transaccional, usa B.
- Si priorizas latencia extrema in-memory, evalua A con mucho cuidado.

---

## 5. Transacciones y ACID

| Propiedad | Que garantiza |
|---|---|
| Atomicidad | Todo o nada |
| Consistencia | Respeta reglas e integridad |
| Aislamiento | Controla interferencia entre transacciones concurrentes |
| Durabilidad | Commit confirmado sobrevive fallas |

### Niveles de aislamiento (SQL)
- Read Uncommitted: max rendimiento, minimo control.
- Read Committed: default comun (PostgreSQL).
- Repeatable Read: evita lecturas no repetibles.
- Serializable: max rigor, menor concurrencia.

### Ejemplo Spring
```kotlin
@Transactional(isolation = Isolation.READ_COMMITTED)
fun transferir(...) { ... }
```

---

## 6. Modelo relacional: lo esencial

### PK y FK
- PK: identifica una fila de forma unica y no nula.
- FK: referencia PK de otra tabla.

### Integridad referencial
Si una fila hija referencia a una padre:
- `ON DELETE RESTRICT`: no deja borrar el padre.
- `ON DELETE CASCADE`: borra hijas automaticamente.
- `ON DELETE SET NULL`: desvincula hijas.

### Ejemplo SQL
```sql
CREATE TABLE pedido (
  id BIGSERIAL PRIMARY KEY,
  cliente VARCHAR(120) NOT NULL
);

CREATE TABLE item_pedido (
  id BIGSERIAL PRIMARY KEY,
  pedido_id BIGINT NOT NULL,
  producto VARCHAR(120) NOT NULL,
  cantidad INT NOT NULL CHECK (cantidad > 0),
  CONSTRAINT fk_item_pedido
    FOREIGN KEY (pedido_id)
    REFERENCES pedido(id)
    ON DELETE CASCADE
);
```

---

## 7. Normalizacion: ejemplo corto

Problema inicial:
- Tabla `pedido` con columnas `producto1`, `producto2`, `producto3`.

Solucion normalizada:
- `pedido` (cabecera)
- `item_pedido` (detalle N)

Beneficio:
- Sin repeticion de columnas.
- Consultas y validaciones mas simples.
- Menor riesgo de inconsistencia.

---

## 8. Docker para laboratorio local

### Comandos base
```bash
docker compose up -d
docker compose down
docker compose down --volumes   # reseteo total
```

### compose minimo (PostgreSQL + pgAdmin)
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: demo
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@demo.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db
```

Tip:
- Entre contenedores usa nombre de servicio (`db`) como host.
- Desde tu maquina usa `localhost:5432`.

---

## 9. Checklist rapido

- Defini PK simple y estable.
- Modela FK segun reglas de negocio.
- Decide estrategia de borrado (`RESTRICT`, `CASCADE`, `SET NULL`).
- Usa transacciones en operaciones de negocio.
- Versiona schema (Flyway en clases siguientes).

---

## 10. Fuentes online

- Spring Transactions: https://docs.spring.io/spring-framework/reference/data-access/transaction.html
- PostgreSQL Isolation Levels: https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL Constraints/FK: https://www.postgresql.org/docs/current/ddl-constraints.html
- Docker Docs: https://docs.docker.com/
- Normalization (overview): https://www.geeksforgeeks.org/normal-forms-in-dbms/
