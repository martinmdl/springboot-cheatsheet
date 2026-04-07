# Clase 1 - Persistencia, Modelo Relacional y Docker

> Material de referencia:
> - [Apunte: Introducción a Persistencia](https://docs.google.com/document/d/1nCy-Xk00lBUrBFQvTWk9P5xsw8ee6JOVklSUlRN3mUI/edit)
> - [Apunte: Modelo Relacional](https://docs.google.com/document/d/1uF3yoYIFmLxTH5ZJoT9I3cc5TW9b-H3BqZJbLudKBcA/edit)
> - [Apunte: Normalización](https://docs.google.com/document/d/1Jil-3oiveXDtY1iKBCof7jE9ooRFJ-f1KjcXgaGk6F0/edit)
> - [Ejercicio: Manejo de Proyectos SQL](https://github.com/uqbar-project/eg-manejo-proyectos-sql)
> - [Taller de Docker](https://docs.google.com/document/d/16-ZVmZQrCbFDDnEyI8eABSp2rwsw3bz1WYyJ7DM9Rxw/edit)
> - [Tutorial Docker - Pelado Nerd](https://www.youtube.com/watch?v=CV_Uf3Dq-EU)

---

## 1. Persistencia

### Qué es el estado de una aplicación

En el paradigma orientado a objetos, el estado de una aplicación vive en los objetos: los valores de sus atributos, las referencias entre ellos, su propia existencia como instancias. Ese estado ocupa memoria RAM asociada al proceso en ejecución.

El problema fundamental: **cuando el proceso termina, la memoria se libera y el estado se pierde**. Eso obliga a pensar en mecanismos que lo hagan sobrevivir entre ejecuciones.

Dos necesidades concretas que surgen de esto:

- Que el estado **sobreviva** a la volatilidad de la memoria (corte de luz, reinicio del servidor, fin del proceso).
- Poder trabajar con estados **más grandes que la memoria disponible**, cargando en RAM solo lo necesario en cada momento.

### Qué significa persistir

**Persistencia** es la capacidad de una aplicación para preservar su estado más allá del ciclo de vida del proceso. Implica dos conceptos:

- **Base de datos:** el conjunto de datos persistidos.
- **Medio persistente:** el soporte físico donde viven esos datos (disco, SSD, cluster, etc.).

Una complicación inmediata: una vez que persistís, existen **dos copias** del objeto — la que vive en memoria y la que duerme en el medio persistente. Mantenerlas sincronizadas es una de las decisiones de diseño centrales de la persistencia.

### Decisiones de diseño al persistir

- ¿Qué datos guardar?
- ¿En qué formato/estructura?
- ¿En qué medio?
- ¿Cuándo guardar y cuándo consultar?
- ¿Cómo mantener la consistencia?

---

## 2. Medios persistentes

Los objetos en memoria forman un **grafo dirigido** (cada objeto es un nodo, cada referencia una arista). Al persistir, hay que mapear esa estructura a algún medio. Las opciones principales:

### Snapshot del ambiente

Una "foto" completa del estado en memoria en un momento dado. Algunos entornos como Smalltalk lo soportan nativamente.

**Limitaciones:** guarda *todo* el ambiente (no solo lo que cambió), tiene alto costo de I/O, y no es útil bajo carga alta o en sistemas multiusuario. Solo viable para aplicaciones monousuario donde el estado no es crítico.

### Serialización a archivos

Se elige un subgrafo de objetos y se lo escribe a un archivo en algún formato. Más eficiente que el snapshot porque solo guarda lo necesario, pero obliga a decidir qué guardar, cuándo y dónde.

**Formatos binarios** (e.g., Java `OutputStream`, Python `pickle`):
- Más rápidos y compactos.
- No legibles por humanos; acoplados a la representación interna del lenguaje/plataforma.

**Formatos de texto** (XML, JSON, YAML):
- Legibles y editables directamente.
- Menos eficientes en espacio y velocidad.
- Limitación estructural: XML/JSON modelan *árboles*, no grafos. Si el grafo tiene ciclos, hay que representarlos con referencias por id.

En el ecosistema Spring/Kotlin usamos **Jackson** (viene integrado en Spring Boot) para serializar a JSON.

### Servidores de base de datos

Se delega en una aplicación especializada que gestiona el almacenamiento, la concurrencia y la consistencia. La comunicación con el servidor se hace mediante su protocolo (SQL para relacionales).

#### Impedance mismatch

El modelo de objetos y el modelo relacional son estructuralmente diferentes. Esa brecha se llama **impedance mismatch**: una unidad cohesiva en objetos (una clase con herencia, una relación muchos-a-muchos) se fragmenta en varias tablas al persistirse. El ORM (Object-Relational Mapper) — Hibernate en nuestro caso — se encarga de este mapeo.

#### Bases de datos de objetos

Eliminan el impedance mismatch almacenando directamente el modelo de objetos. Soportan herencia, referencias polimórficas y relaciones circulares. Sin embargo, no son la solución dominante en la industria; el modelo relacional sigue siendo el estándar.

---

## 3. Estrategias de persistencia

### Ambiente vivo + medio persistente como backup

Todos los objetos viven en memoria. Cada cambio se sincroniza con la BD (sincrónica o asincrónicamente). El dueño de los datos es el modelo de objetos; la BD es solo un respaldo.

**Problema principal:** la BD solo puede garantizar transaccionalidad sobre los datos que *ella* maneja. Si ocurre una excepción en el ambiente después de que algunos cambios ya se aplicaron en memoria pero antes de persistirse, el estado queda inconsistente. Esto requiere implementar lógica de rollback manual sobre los objetos en memoria.

**Variante — Esquema prevalente (Prevayler):** todas las operaciones con efecto colateral se modelan como comandos (patrón Command) y se persisten en un log. A intervalos amplios se hace un snapshot completo. Al recuperarse de una falla: snapshot + reprocesamiento del log. Alta performance (todo en RAM), pero requiere un único ambiente principal.

### Ambiente muerto / recreado ante cada estímulo

El ambiente arranca vacío. Ante cada request, reconstruye el estado necesario desde la BD, aplica los cambios y los persiste nuevamente. Es el modelo que usa Spring Boot con JPA.

**Ventaja:** se aprovecha completamente la transaccionalidad del servidor de BD — si hay una excepción, el rollback descarta tanto los cambios en la BD como los objetos en memoria (que de todas formas se descartan siempre).

**Costo:** pérdida de identidad de objeto entre requests; la misma entidad lógica es representada por instancias diferentes en cada request.

---

## 4. Transaccionalidad y propiedades ACID

Una **transacción** es un conjunto de operaciones que, desde el punto de vista del negocio, deben tratarse como una única unidad de trabajo: o se completan todas, o no se completa ninguna.

Las cuatro propiedades ACID que debe garantizar una transacción:

**Atomicidad:** todo o nada. Si algún paso falla, se deshacen todos los cambios ya aplicados.

**Consistencia:** la transacción lleva la BD de un estado válido a otro estado válido, respetando todas las restricciones de integridad y reglas de negocio.

**Aislamiento:** los efectos de una transacción no son visibles para otras transacciones hasta que se hace commit. Esto se vuelve crítico en sistemas concurrentes. Existen distintos **niveles de aislamiento** (Read Uncommitted, Read Committed, Repeatable Read, Serializable) que balancean consistencia vs. performance — a mayor aislamiento, mayor costo.

**Durabilidad:** una vez confirmada (commit), la transacción persiste aunque el sistema falle inmediatamente después.

> Las propiedades ACID no están escritas a fuego. En sistemas distribuidos o de alta carga, algunas se relajan conscientemente (especialmente el aislamiento) a cambio de mayor escalabilidad. Entender esas compensaciones es parte del diseño arquitectural.

Los servidores de BD exponen tres operaciones para gestionar transacciones: **BEGIN**, **COMMIT** y **ROLLBACK**.

---

## 5. Modelo relacional

> Apunte completo: [Diseño de datos: Modelo Relacional](https://docs.google.com/document/d/1uF3yoYIFmLxTH5ZJoT9I3cc5TW9b-H3BqZJbLudKBcA/edit)

El modelo relacional fue propuesto por Edgar F. Codd en 1970. Su idea central: representar *toda* la información como tablas (relaciones) de filas y columnas.

### Conceptos clave

**Relación (tabla):** estructura con una cabecera fija (conjunto de atributos) y un cuerpo variable (conjunto de tuplas). Importante: una relación *no* tiene orden de filas ni de columnas — la tabla es solo una representación visual.

**Dominio:** conjunto de valores posibles para un atributo. Los dominios son atómicos (no se descomponen). Permiten restringir comparaciones semánticamente inválidas (comparar un legajo con un código de materia no tiene sentido aunque ambos sean enteros).

**Propiedades formales de una relación:**
- No hay tuplas repetidas.
- No hay orden entre las tuplas ni entre los atributos.
- Todos los valores son atómicos (primera forma normal implícita).

### Claves

**Clave candidata:** atributo o conjunto mínimo de atributos que identifica unívocamente cada tupla (unicidad + minimalidad).

**Clave primaria (PK):** la clave candidata elegida por el diseñador. Nunca puede ser nula — si no podés identificar algo, no podés representarlo en la BD.

**Clave alternativa:** claves candidatas no elegidas como primaria (e.g., CUIT o combinación tipo+número de documento, además del id de cliente).

**Clave foránea (FK):** atributo en una tabla R2 cuyos valores deben existir como PK en otra tabla R1. Es el mecanismo de referencia entre tablas. Puede ser nula si la relación es opcional.

### Reglas de integridad

**Integridad de entidades:** ningún componente de la PK puede ser nulo. No podés guardar algo que no podés identificar.

**Integridad referencial:** cada valor de FK debe existir como PK en la tabla referenciada (o ser nulo si se permite). Si querés eliminar un registro referenciado, tenés tres opciones: RESTRICT (no se permite), CASCADE (se eliminan los hijos), o ANULACIÓN (se pone null en las FK).

**Reglas de negocio:** restricciones específicas de cada organización, implementables en el motor mediante CHECK constraints o TRIGGERS.

---

## 6. Normalización

> Apunte completo: [Diseño de datos: Normalización](https://docs.google.com/document/d/1Jil-3oiveXDtY1iKBCof7jE9ooRFJ-f1KjcXgaGk6F0/edit)

Normalizar es transformar un diseño con redundancias en una estructura donde los datos son consistentes y fáciles de mantener. El objetivo no es seguir mecánicamente un algoritmo, sino entender qué problemas resuelve cada forma normal.

**Regla general:** todo determinante debe ser clave candidata.

### Dependencia funcional

`A → B` significa que dado un valor de A, siempre hay un único valor de B asociado. A es *determinante* de B.

### Forma canónica

Eliminar atributos calculables (totales, conteos, promedios que se pueden derivar de otros datos). Guardar datos calculados genera redundancia y riesgo de inconsistencia.

### Primera Forma Normal (1FN)

No hay grupos repetitivos ni atributos multivaluados. Cada celda tiene un único valor atómico. Si un pedido puede tener N gustos de pizza, no podés poner todos los gustos en una sola columna — necesitás una tabla aparte.

### Segunda Forma Normal (2FN)

Cumple 1FN y no hay dependencias parciales: ningún atributo no-clave depende de *parte* de una clave compuesta. Si en `Pedido_Gusto(id_pedido, id_gusto, cantidad, descripcion_gusto)` la descripción del gusto depende solo de `id_gusto` (no de toda la clave compuesta), hay que separar `Gustos` en su propia tabla.

### Tercera Forma Normal (3FN)

Cumple 2FN y no hay dependencias transitivas: ningún atributo no-clave depende de otro atributo no-clave. Si en `Pedido` el nombre y domicilio del cliente dependen del `id_cliente` (que no es PK de Pedido), hay que mover esos datos a una tabla `Clientes`.

### Forma Normal de Boyce-Codd (BCNF / 3.5FN)

Extiende 3FN a un caso borde: cuando hay claves candidatas compuestas que se solapan. Todo determinante debe ser clave candidata, incluso si no forma parte de la PK elegida.

### 4FN y 5FN

Atacan dependencias multivaluadas en relaciones muchos-a-muchos. En la práctica se usan menos, pero el principio es el mismo: eliminar redundancia separando relaciones independientes en tablas propias.

### Desnormalización — cuándo romper las reglas a propósito

Normalizar tiene costos: los JOINs entre muchas tablas pueden ser lentos bajo alto volumen. A veces se desnormaliza deliberadamente para ganar performance:

- **Vistas materializadas:** copias físicas del resultado de un JOIN, actualizadas periódicamente.
- **Campos redundantes:** guardar datos de otra tabla (e.g., el domicilio de entrega en el pedido) para tener un historial o evitar JOINs frecuentes.

> La desnormalización siempre es una decisión consciente y justificada. Hay que saber qué redundancia se acepta y por qué.

---

## 7. Ejercicio: Manejo de Proyectos con componentes en la BD

> Repo: [eg-manejo-proyectos-sql](https://github.com/uqbar-project/eg-manejo-proyectos-sql)

El ejercicio muestra cómo resolver lógica de negocio usando componentes del motor de la base de datos: stored procedures, funciones, triggers y constraints.

**Para levantar el entorno:**

```bash
docker compose up
```

Esto levanta dos contenedores:
- **PostgreSQL** en el puerto 5432 (usuario y contraseña: `postgres`)
- **pgAdmin** en `http://localhost:5050/` (usuario: `admin@phm.edu.ar`, contraseña: `admin`)

Al configurar el server en pgAdmin, el Host Name es el nombre del contenedor de Postgres definido en `docker-compose.yml` (en este caso `manejo_proyectos_sql`), no `localhost`.

**Para regenerar la BD desde cero:**

```bash
docker compose down --volumes
docker compose up
```

**Estructura de los scripts SQL** — los archivos tienen prefijos numéricos para garantizar el orden de ejecución:

| Archivo | Contenido |
|---|---|
| `01_init_db.sql` | Creación de la base y roles |
| `10_..._DDL.sql` | Definición de tablas (DDL) |
| `20_..._Fixture.sql` | Datos de prueba |
| `30_..._Pruebas.sql` | Queries de ejemplo |
| `51-56_...sql` | Componentes: overhead, subtareas, costo, log, restricciones |

**Componentes del motor que aparecen en el ejemplo:**

- **Stored procedures / funciones:** lógica encapsulada en la BD, ejecutable con una llamada SQL.
- **Triggers:** lógica que se dispara automáticamente ante eventos (INSERT, UPDATE, DELETE). En el ejemplo se usa para loguear inserciones de tareas.
- **CHECK constraints:** restricciones declarativas sobre columnas (e.g., que la descripción de una tarea no esté vacía, que el tiempo sea positivo).

> **Reflexión de diseño:** poner lógica en la BD tiene ventajas (aplica para cualquier cliente que acceda a la BD) pero complica el testing y el versionado. En la materia vamos a ver cómo Flyway nos ayuda a versionar estos scripts igual que versionamos el código.

---

## 8. Docker

> Taller completo: [Taller de introducción a Docker](https://docs.google.com/document/d/16-ZVmZQrCbFDDnEyI8eABSp2rwsw3bz1WYyJ7DM9Rxw/edit)
> Video recomendado: [Tutorial Pelado Nerd](https://www.youtube.com/watch?v=CV_Uf3Dq-EU)

### El problema que resuelve

"En mi máquina funciona." Docker soluciona el problema de compatibilidad de dependencias entre entornos de desarrollo, staging y producción, estandarizando la unidad de despliegue: el **contenedor**.

La analogía es el contenedor de shipping: puertos, grúas, barcos y camiones están todos diseñados para las mismas dimensiones. Docker hace lo mismo con el software.

### Contenedor vs. VM

Un contenedor no es una VM completa. No tiene un OS propio — comparte el kernel del host y solo empaqueta los binarios y bibliotecas que necesita la aplicación. Esto lo hace órdenes de magnitud más liviano (cientos de MB vs. decenas de GB) y más rápido de iniciar.

**Cómo funciona internamente (Linux):**
- **chroot:** confina el proceso a un subárbol del sistema de archivos.
- **cgroups y namespaces:** limitan los recursos visibles (CPU, memoria, red) — el contenedor no sabe que existen recursos fuera de su cuota.
- **UnionFS:** permite montar un directorio del host en el contenedor, compartiendo archivos entre ambos contextos.

### Imágenes y capas

Una **imagen** es la plantilla inmutable a partir de la cual se crean contenedores. La relación imagen-contenedor es análoga a clase-instancia en OOP.

Las imágenes están compuestas por **capas inmutables**, donde cada capa aporta un delta sobre la anterior (similar a los commits de git). Esto permite reutilizar capas entre imágenes — si dos imágenes comparten capas de Ubuntu, esas capas se almacenan una sola vez.

`docker run` **siempre crea un nuevo contenedor** a partir de la imagen. Los cambios dentro del contenedor no modifican la imagen base.

### Comandos esenciales

```bash
# Correr un contenedor (descarga la imagen si no está local)
docker run hello-world

# Contenedor interactivo
docker run -it ubuntu:22.04

# Ver contenedores activos
docker ps

# Ver todos los contenedores (incluyendo detenidos)
docker ps -a

# Reiniciar un contenedor detenido
docker restart <nombre_o_id>

# Ejecutar un comando en un contenedor corriendo
docker exec -it <nombre> /bin/bash

# Ver imágenes locales
docker images
```

### Dockerfile

Un `Dockerfile` es la receta para construir una imagen. Se ejecuta en tiempo de build, no en tiempo de ejecución:

```dockerfile
FROM ubuntu:22.04          # imagen base
RUN apt-get update         # se ejecuta en BUILD, agrega una capa
COPY . /app                # copia archivos del contexto al contenedor
WORKDIR /app
CMD ["node", "index.js"]   # comando por defecto al correr el contenedor
```

```bash
docker build -t mi-imagen:latest .
```

El `.` es el *contexto de build* — todos los archivos en ese directorio son accesibles durante el build. Usar `.dockerignore` para excluir lo que no se necesita (`node_modules`, `target/`, etc.).

### Networking y puertos

Los contenedores están aislados por defecto — no exponen puertos al exterior. Para mapear un puerto del contenedor al host:

```bash
docker run -p 5432:5432 postgres   # host_port:container_port
```

Para que dos contenedores se comuniquen entre sí, se los conecta a una red virtual de Docker:

```bash
docker network create mi-red
docker network connect mi-red contenedor1
docker network connect mi-red contenedor2
```

Dentro de la misma red, cada contenedor es accesible por su **nombre** como hostname (no por IP). Eso es lo que permite que en `docker-compose.yml` el servicio `app` acceda al servicio `db` usando `db` como hostname.

### Volúmenes — persistencia de datos

Los datos dentro de un contenedor se pierden cuando el contenedor se elimina. Para persistir datos (e.g., la BD de PostgreSQL), se monta un directorio del host dentro del contenedor:

```bash
docker run -v /ruta/local:/var/lib/postgresql/data postgres
```

Cualquier cambio en `/var/lib/postgresql/data` dentro del contenedor se refleja en `/ruta/local` del host, y viceversa. Así, al recrear el contenedor con el mismo volumen, los datos sobreviven.

### Docker Compose

Docker Compose orquesta múltiples contenedores desde un archivo `docker-compose.yml`. En lugar de ejecutar varios `docker run` con flags complejos, se declaran los servicios:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@phm.edu.ar
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db

volumes:
  postgres_data:
```

```bash
docker compose up          # levanta todos los servicios (crea si no existen)
docker compose up -d       # en background (detached)
docker compose down        # detiene y elimina contenedores y redes
docker compose down --volumes  # también elimina los volúmenes (resetea la BD)
docker compose start       # levanta contenedores ya creados
```

**`depends_on`** asegura el orden de inicio: en el ejemplo, pgAdmin arranca después de que la BD esté lista.

Por defecto, todos los servicios de un `docker-compose.yml` están en la misma red y se ven entre sí por nombre de servicio.

### Registry — Docker Hub

Las imágenes se distribuyen a través de registries. Docker Hub es el público por defecto. Al hacer `docker run postgres`, Docker busca la imagen `postgres` en Docker Hub si no la tiene localmente.

```bash
docker pull postgres:16-alpine    # descarga sin correr
docker push mi-usuario/mi-imagen  # sube una imagen propia (requiere login)
```

Al subir, solo se envían las capas que no existen ya en el registry — las capas compartidas con imágenes oficiales (Ubuntu, Alpine) no se re-suben.

---

## Conexión entre los temas de la clase

La secuencia lógica de la materia parte de aquí:

1. **Persistencia** define el problema: el estado de los objetos debe sobrevivir al proceso.
2. **El modelo relacional** es la solución dominante en la industria para almacenar ese estado.
3. **La normalización** da criterios para diseñar ese modelo correctamente.
4. **Docker** es la herramienta que nos permite tener un entorno de BD reproducible y consistente entre todos los desarrolladores del equipo — sin Docker, "en mi máquina funciona" vuelve a ser un problema.
5. En las clases siguientes, **JPA/Hibernate** será el puente entre el modelo de objetos y el modelo relacional, automatizando el mapeo que hace el ORM.