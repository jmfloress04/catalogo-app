## Etapa 1 — Arquitectura

    navegador ──► catalogo-frontend ──────► catalogo-api ──────► catalogo-db
                  React + Vite              Python + FastAPI      MongoDB 7
                  nginx :8080               :8000                 :27017
                  (host 3000)               (host 8000, solo      (sin puerto
                                             depuración)           publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `catalogo-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `catalogo-api` | Python + FastAPI | 8000 | 8000 (solo depuración) |
| `catalogo-db` | MongoDB 7 | 27017 | — |

El frontend es el único punto de entrada: nadie le habla a la base
directamente, y a la API le habla el frontend.

## Etapa 2 — Medición y Optimización de Imágenes

| Imagen | Tag | Imagen Base Final | Tamaño en disco | Tamaño comprimido | Reducción |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `catalogo-api` (Ingenua) | `ingenua` | `python:3.12` | 1,72 GB | 447 MB | — |
| `catalogo-api` (Definitiva) | `v1` | `python:3.12-slim` (Multi-stage) | 248 MB | 59,1 MB | **~86 % menos** |
| `catalogo-frontend` | `v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | 73,9 MB | 21 MB | **~96 % menos** (vs API ingenua) |

> *Tamaño en disco* = columna DISK USAGE de `docker images` (imagen descomprimida).
> *Tamaño comprimido* = columna CONTENT SIZE (lo que se descarga del registry).

### Qué aportó cada decisión:
1. **Cambio de imagen base a `-slim` / `-alpine` + Multi-etapa:** Entre ambas eliminaron ~1,47 GB respecto de la versión ingenua (1,72 GB → 248 MB): herramientas del sistema operativo, compiladores C y librerías de desarrollo innecesarias en ejecución.
2. **Construcción Multi-etapa (`Multi-stage Build`):** Permitió dejar afuera de la imagen final la caché de `pip`/`npm`, el toolchain de Node.js (`node_modules`) y el código fuente original sin compilar.
3. **Uso de `.dockerignore`:** Evitó transferir `.git`, `.venv` y `node_modules` locales al daemon de Docker y evitó filtrar archivos `.env`.
4. **Ejecución sin privilegios (`USER appuser` / `nginx-unprivileged`):** Redujo la superficie de ataque impidiendo que el proceso corra como `root` dentro del contenedor.

### Análisis de capas (`docker history`)

Comando utilizado (filtrando las capas de 0B, que son solo metadatos):

```bash
docker history --format "table {{.ID}}\t{{.CreatedBy}}\t{{.Size}}" catalogo-api:v1 | grep -v "0B"
```

Salida:

```
IMAGE          CREATED BY                                      SIZE
<missing>      COPY --chown=appuser:appuser . . # buildkit     553kB
<missing>      COPY --chown=appuser:appuser /root/.local /h…   54MB
<missing>      RUN /bin/sh -c useradd --create-home --shell…   69.6kB
<missing>      WORKDIR /app                                    8.19kB
<missing>      RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB
<missing>      RUN /bin/sh -c set -eux;   savedAptMark="$(a…   41.4MB
<missing>      RUN /bin/sh -c set -eux;  apt-get update;  a…   4.92MB
<missing>      # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.7MB
```

**Capa más pesada:** `debian.sh ... 'trixie'` (87,7 MB). Corresponde al sistema operativo base (Debian 13 mínimo) de la imagen `python:3.12-slim`. Junto con la instalación de Python (41,4 MB) y las librerías del sistema (4,92 MB), la imagen base aporta ~134 MB, que no se pueden reducir sin cambiar de imagen base.

**Capa más pesada propia:** `COPY --from=builder /root/.local` (54 MB). Son las dependencias de `requirements.txt` (FastAPI, uvicorn, driver de MongoDB) instaladas en la etapa builder con `pip install --user`. El código de la aplicación (`main.py`, `catalogo_demo.py` e `imagenes/`) pesa solo 553 kB.

**Evidencia del multi-etapa:** no aparecen las capas de la etapa builder (`WORKDIR /build`, `pip install`). La imagen final solo contiene lo traído con `COPY --from=builder`.

### Efecto del cache de construcción

**Procedimiento.** Para no sobrescribir la imagen `catalogo-api:v1` ya publicada, el experimento se hizo con un tag aparte, `catalogo-api:cache-test`. El cache de construcción es el mismo sin importar el tag.

```bash
# 0. Build inicial (cache cargado)
docker build --progress=plain -t catalogo-api:cache-test ./backend

# 1. Modificar código y reconstruir
echo "# prueba de cache" >> backend/main.py
docker build --progress=plain -t catalogo-api:cache-test ./backend

# 2. Modificar requirements.txt y reconstruir
echo "# prueba de cache" >> backend/requirements.txt
docker build --progress=plain -t catalogo-api:cache-test ./backend
```

`--progress=plain` muestra la salida completa de cada paso, donde se lee si salió `CACHED` o se volvió a ejecutar.

**Prueba 1: modificación de código** (`backend/main.py`)

```
#9 [builder 3/4] COPY requirements.txt .
#9 CACHED

#10 [builder 4/4] RUN pip install --user --no-cache-dir -r requirements.txt
#10 CACHED

#11 [stage-1 4/5] COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
#11 CACHED

#12 [stage-1 5/5] COPY --chown=appuser:appuser . .
#12 DONE 0.2s
```

**Prueba 2: modificación de `requirements.txt`**

```
#7 [builder 3/4] COPY requirements.txt .
#7 DONE 0.1s

#8 [builder 4/4] RUN pip install --user --no-cache-dir -r requirements.txt
#8 3.729 Collecting fastapi==0.115.0 (from -r requirements.txt (line 1))
#8 3.942 Collecting uvicorn==0.30.6 (from uvicorn[standard]==0.30.6->-r requirements.txt (line 2))
#8 4.445 Collecting pymongo==4.8.0 (from -r requirements.txt (line 3))
...
#8 11.89 Successfully installed annotated-types-0.8.0 anyio-4.16.0 ... uvicorn-0.30.6 ...
#8 DONE 12.8s

#11 [stage-1 4/5] COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
#11 DONE 0.6s

#12 [stage-1 5/5] COPY --chown=appuser:appuser . .
#12 DONE 0.2s
```

| Cambio | `pip install` | Tiempo de `pip install` |
| :--- | :---: | :---: |
| Código (`main.py`) | CACHED | 0 s |
| `requirements.txt` | se reejecuta (descarga e instala 20 paquetes) | 12,8 s |

**Explicación:** Docker reutiliza cada capa mientras su instrucción y los archivos que copia no cambien; cuando una capa se invalida, todas las siguientes se reconstruyen en cascada (en la prueba 2, `COPY . .` se rehízo aunque el código no cambió). Como `requirements.txt` se copia e instala **antes** que el código, modificar el código solo invalida el último `COPY`, mientras que modificar el manifiesto obliga a reinstalar todas las dependencias.

**Limpieza.** Al terminar, se revirtieron los cambios y se eliminó la imagen de prueba:

```bash
git restore backend/main.py backend/requirements.txt
docker rmi catalogo-api:cache-test
```

## Etapa 3 — Redes y Almacenamiento

### Red y volumen

```bash
docker network create catalogo-net
docker volume create catalogo-db-data
```

### Base de datos (`catalogo-db`)

Comando utilizado para levantar MongoDB 7 conectado a `catalogo-net`, con el volumen nombrado en `/data/db` y **sin publicar puertos**:

```bash
docker run -d \
  --name catalogo-db \
  --network catalogo-net \
  -e MONGO_INITDB_ROOT_USERNAME=catalogo_user \
  -e MONGO_INITDB_ROOT_PASSWORD=catalogo_pass \
  -v catalogo-db-data:/data/db \
  mongo:7
```

Resultado (`docker ps --filter name=catalogo-db`):

```text
CONTAINER ID   IMAGE     COMMAND                  CREATED        STATUS          PORTS       NAMES
8ea8ea4c7622   mongo:7   "docker-entrypoint.s…"   14 hours ago   Up 10 minutes   27017/tcp   catalogo-db
```

*(En la columna `PORTS` aparece únicamente `27017/tcp`, sin mapeo `0.0.0.0:27017->27017/tcp`, lo que confirma que el puerto no fue publicado en el anfitrión).*

Línea del log que confirma que el motor acepta conexiones (`docker logs catalogo-db | grep -i "Waiting for connections"`):

```json
{"t":{"$date":"2026-11-22T11:55:38.094+00:00"},"s":"I",  "c":"NETWORK",  "id":23016,   "ctx":"listener","msg":"Waiting for connections","attr":{"port":27017,"ssl":"off"}}
```

**¿Por qué no se publica el puerto (`-p`) de la base de datos?**
Porque `catalogo-db` es la capa más interna y sensible de la arquitectura de tres capas. Al omitir `-p`, el puerto `27017` no queda expuesto ni al sistema anfitrión ni a la red externa, reduciendo la superficie de ataque: únicamente los contenedores conectados a la red privada `catalogo-net` (en este caso, `catalogo-api`) pueden alcanzarla por nombre. Si se necesita administración manual desde el host, se accede directamente mediante `docker exec -it catalogo-db mongosh`.

### API (`catalogo-api`) y resolución DNS interna

Comando utilizado para levantar la API en `catalogo-net` alcanzando la base por nombre (`DB_HOST=catalogo-db`):

```bash
docker run -d \
  --name catalogo-api \
  --network catalogo-net \
  -p 8000:8000 \
  -e DB_HOST=catalogo-db \
  -e DB_PORT=27017 \
  -e DB_NAME=catalogodb \
  -e DB_USER=catalogo_user \
  -e DB_PASSWORD=catalogo_pass \
  -e APP_VERSION=v1 \
  -e ROOT_PATH=/api \
  -e SEED_DEMO=true \
  catalogo-api:v1
```

Comprobaciones:

1. **Resolución DNS interna:**
   ```bash
   docker exec catalogo-api getent hosts catalogo-db
   ```
   ```text
   172.18.0.2      catalogo-db
   ```
2. **Estado de salud:**
   ```bash
   curl -s localhost:8000/health
   ```
   ```json
   {"status":"ok","db":"ok","version":"v1"}
   ```
3. **Productos sembrados (`SEED_DEMO=true`):**
3. ```bash
   curl -s localhost:8000/productos | python3 -c "import sys,json; d=json.load(sys.stdin); print('Total productos:', len(d))"
   ```
   ```text
   Total productos: 12
   ```

*(El valor `DB_HOST=catalogo-db` es un nombre de contenedor, no una IP; lo resuelve el servidor DNS interno `127.0.0.11` de la red `bridge` custom `catalogo-net`. En la red `bridge` predeterminada esta resolución no funcionaría).*

### Diagnóstico de fallas: resolución de nombre vs. autenticación

1. **Fallo de resolución de nombre (`DB_HOST=catalogo-bd`):**
   ```text
   [2026-11-22T13:10:29+0000] WARNING intento 1/10 de conexión falló: catalogo-bd:27017: [Errno -3] Temporary failure in name resolution ...
   ...
   pymongo.errors.ServerSelectionTimeoutError: catalogo-bd:27017: [Errno -3] Temporary failure in name resolution
   ERROR:    Application startup failed. Exiting.
   ```

2. **Fallo de autenticación (`mongosh` sin `?authSource=admin`):**
   ```bash
   docker exec catalogo-db mongosh "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogodb" \
     --eval "db.productos.countDocuments()"
   ```
   ```text
   MongoServerError: Authentication failed.
   ```

**¿Por qué el primero indica que NO se alcanzó la base y el segundo que SI se la alcanzó?**
El primer error (`[Errno -3] Temporary failure in name resolution`) ocurre en la capa de red/DNS antes de abrir siquiera una conexión TCP, porque el DNS interno de Docker no conoce ningún host llamado `catalogo-bd`. En cambio, `MongoServerError: Authentication failed` es una respuesta emitida por el propio motor MongoDB tras recibir la conexión y rechazar las credenciales al buscarlas en `catalogodb` en lugar de `admin` (por omitir `?authSource=admin`), lo que confirma que la base sí fue alcanzada.

### Fallo del frontend fuera de la red (`catalogo-frontend`)

Al ejecutar `catalogo-frontend` omitiendo `--network catalogo-net`, el contenedor cae en la red `bridge` predeterminada (sin DNS interno):

```bash
docker run -d \
  --name catalogo-frontend \
  -p 3000:8080 \
  -e API_HOST=catalogo-api \
  -e API_PORT=8000 \
  catalogo-frontend:v1
```

Estado (`docker ps -a --filter name=catalogo-frontend`):

```text
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS                      PORTS     NAMES
7340873abafb   catalogo-frontend:v1   "/docker-entrypoint.…"   14 seconds ago   Exited (1) 12 seconds ago             catalogo-frontend
```

Mensaje del registro (`docker logs catalogo-frontend`):

```text
20-envsubst-on-templates.sh: Running envsubst on /etc/nginx/templates/default.conf.template to /etc/nginx/conf.d/default.conf
...
2026/11/22 13:48:42 [emerg] 1#1: host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
nginx: [emerg] host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
```

**¿Por qué el proceso termina (`Exited (1)`) en lugar de arrancar en estado degradado?**
Nginx resuelve por DNS los nombres configurados en `proxy_pass` una única vez al iniciar el proceso. Como el contenedor se inició fuera de `catalogo-net` (en la red bridge por defecto sin DNS interno), no puede resolver `catalogo-api` y no reintenta: aborta inmediatamente la inicialización y el contenedor muere con código de salida `1`(`Exited (1)`).

### Frontend conectado a `catalogo-net` y verificación del reverse proxy

Se eliminó el contenedor detenido y se volvió a crear utilizando **exactamente la misma imagen** (`catalogo-frontend:v1`, sin reconstruir nada), agregando únicamente `--network catalogo-net`:

```bash
docker rm catalogo-frontend

docker run -d \
  --name catalogo-frontend \
  --network catalogo-net \
  -p 3000:8080 \
  -e API_HOST=catalogo-api \
  -e API_PORT=8000 \
  catalogo-frontend:v1
```

**Observación en DevTools (pestaña `Network`) al cargar un producto con imagen en `http://localhost:3000`:**
- La solicitud `POST` para crear el producto (y las consultas `GET` del catálogo) se dirigen a **`http://localhost:3000/api/productos`** —es decir, sobre el **mismo origen** (`http://localhost:3000`) que sirvió el SPA— y **no** a `http://localhost:8000`.
- **Ausencia de solicitudes `OPTIONS`:** No se dispara ningún *preflight* `OPTIONS` de CORS porque, al compartir protocolo, host y puerto, el navegador considera que la petición es *same-origin*. Es Nginx quien actúa como *reverse proxy*, recortando el prefijo `/api` y reenviando internamente la petición a `catalogo-api:8000` dentro de `catalogo-net`.

### Verificación del proxy y efecto de `ROOT_PATH` en FastAPI

1. **Comparación de acceso directo vs. a través del proxy:**
   `curl -s localhost:8000/productos` y `curl -s localhost:3000/api/productos` devuelven exactamente el mismo listado JSON de productos, confirmando que Nginx recorta el prefijo `/api` gracias a la barra final de `proxy_pass http://${API_HOST}:${API_PORT}/;` y reenvía `/productos` a la API.

2. **`openapi.json` con `ROOT_PATH=/api`:**
   ```bash
   curl -s localhost:8000/openapi.json | python3 -m json.tool | head -n 15
   ```
   ```json
   {
       "openapi": "3.1.0",
       "info": {
           "title": "catalogo-api",
           "description": "Catálogo de productos de la Variante C del proyecto integrador.",
           "version": "v1"
       },
       "servers": [
           {
               "url": "/api"
           }
       ],
       "paths": {
   ```

3. **`openapi.json` sin prefijo (`ROOT_PATH=""`):**
   *(Nota: en `backend/main.py:42`, `ROOT_PATH = os.environ.get("ROOT_PATH", "/api")` define `"/api"` por defecto cuando la variable se omite, por lo que para desactivar el prefijo se pasó explícitamente `-e ROOT_PATH=""`).*
   ```json
   {
       "openapi": "3.1.0",
       "info": {
           "title": "catalogo-api",
           "description": "Catálogo de productos de la Variante C del proyecto integrador.",
           "version": "v1"
       },
       "paths": {
   ```
   La clave `"servers"` desaparece por completo del documento.

4. **Diferencia observada en `http://localhost:3000/api/docs`:**
   - **Con `ROOT_PATH=/api`:** Swagger UI carga normalmente y construye sus rutas anteponiendo `/api` (obtiene el esquema desde `/api/openapi.json` y ejecuta las pruebas contra `/api/...` a través de Nginx).
   - **Sin prefijo (`ROOT_PATH=""`):** Al desaparecer `servers` y no conocer el prefijo `/api`, Swagger UI intenta cargar `/openapi.json` desde la raíz (`http://localhost:3000/openapi.json`). Nginx no matchea `location /api/` y sirve el `index.html` del SPA en lugar de JSON, haciendo que la documentación falle / no cargue a través del proxy.

### Persistencia de datos en MongoDB (`catalogo-db-data`)

Se destruyó el contenedor de la base de datos (`docker rm -f catalogo-db`), se verificó con `docker volume ls` que el volumen `catalogo-db-data` permanecía intacto, se recreó `catalogo-db` montando `-v catalogo-db-data:/data/db` y se reinició la API (`docker restart catalogo-api`).

Verificación del conteo de documentos tras recrear la base:

```bash
docker exec catalogo-db mongosh "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogodb?authSource=admin" \
  --quiet --eval "db.productos.countDocuments()"
```

```text
13
```

El producto cargado desde el navegador en el paso 6 continuó disponible **con su imagen y con su identificador original**, porque los bytes de la imagen se almacenan dentro del documento en MongoDB (en el volumen `catalogo-db-data`) y no en el disco efímero del contenedor `catalogo-api`. Esto mantiene a la API sin estado (*stateless*) y permite escalarla a múltiples réplicas en Kubernetes (Semana 6).

### Orden de arranque manual

Para levantar `catalogo-app` con `docker run` debe respetarse estrictamente este orden:

**`red (catalogo-net)` → `volumen (catalogo-db-data)` → `base (catalogo-db)` → `API (catalogo-api)` → `frontend (catalogo-frontend)`**

**Qué falla al invertir cada paso:**
- **Contenedor antes que la red:** `docker run --network catalogo-net` falla porque la red no existe (o, si se omite `--network`, el contenedor cae en `bridge` por defecto y no resuelve nombres por DNS).
- **Base de datos sin el volumen nombrado:** los datos se guardan en la capa de escritura efímera del contenedor y se pierden definitivamente con `docker rm`.
- **API antes que la base de datos:** la API no logra conectar con MongoDB al arrancar (reintentos fallidos en `esperar_mongo()` hasta agotar intentos y salir con error).
- **Frontend antes que la API:** Nginx resuelve el *upstream* `catalogo-api` una única vez al iniciar; si aún no existe en el DNS de `catalogo-net`, no reintenta y muere de inmediato (`Exited (1)`) con `nginx: [emerg] host not found in upstream "catalogo-api"`.

**Limitación actual (resuelta en la Semana 4):**
Todo este orden de arranque, las variables de entorno, la red y el volumen residen hoy únicamente en la memoria del operador (o como comandos sueltos en este `README.md`) y no en ningún archivo ejecutable. En la Semana 4, **Docker Compose** (`docker-compose.yml`) reemplaza esta secuencia manual por una definición declarativa reproducible con un solo comando.

## Etapa 4 — Docker Compose y Seguridad Básica

### Healthcheck de `catalogo-db` (MongoDB 7)

En `docker-compose.yml` el servicio `catalogo-db` declara su chequeo de salud con `mongosh` (ya incluido en la imagen oficial `mongo:7`):

```yaml
healthcheck:
  test:
    - CMD-SHELL
    - mongosh --quiet --eval "db.adminCommand('ping').ok" || exit 1
  interval: 5s
  timeout: 3s
  retries: 10
  start_period: 10s
```

- **Por qué no lleva credenciales:** `db.adminCommand('ping')` es uno de los pocos comandos administrativos de MongoDB que **no requieren autenticación**. Solo verifica que el proceso `mongod` esté levantado y aceptando comandos en el puerto `27017`.
- **Por qué es una ventaja frente a la Variante B (MySQL):** En motores donde el healthcheck requiere autenticarse (como `mysqladmin ping` con usuario y contraseña), las credenciales deben viajar dentro del propio comando del healthcheck, lo que aumenta el riesgo de exponerlas en `docker inspect`, logs o la lista de procesos del contenedor. Con `mongosh` + `ping`, ninguna contraseña forma parte del comando.
- **Qué NO verifica (riesgo asociado):** Que `ping` responda `ok` **no prueba que el usuario `catalogo_user` exista ni que la contraseña configurada sea la correcta**. Si el volumen `catalogo-db-data` ya contenía una base inicializada con otra contraseña (recordando que `MONGO_INITDB_ROOT_*` solo se aplica sobre un `/data/db` vacío), `catalogo-db` puede figurar como `healthy` y aun así `catalogo-api` fallará con `Authentication failed`. Un healthcheck verifica únicamente lo que prueba, no que toda la configuración sea válida.

### Healthcheck de `catalogo-api` (FastAPI)

El servicio `catalogo-api` verifica su estado usando únicamente la biblioteca estándar de Python (`urllib.request`), sin agregar ningún paquete externo a la imagen:

```yaml
healthcheck:
  test:
    - CMD
    - python
    - -c
    - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"
  interval: 5s
  timeout: 3s
  retries: 10
  start_period: 15s
```

- **Por qué no corresponde instalar `curl` en la imagen solo para el healthcheck:**
  1. **Pierde la ventaja de usar imágenes livianas (`slim` / `alpine`):** Instalar `curl` (junto con sus dependencias como `libcurl`, `openssl` y utilidades adicionales) aumenta el tamaño de la imagen `python:3.12-slim` construida con *multi-stage build*.
  2. **Aumenta la superficie de ataque:** Cada binario y librería adicional instalada en la imagen implica más paquetes susceptibles a vulnerabilidades (CVE) en los escaneos de seguridad.
  3. **Es innecesario:** `urllib.request` ya viene incluido en la biblioteca estándar de Python. Si `http://127.0.0.1:8000/health` responde HTTP `2xx`, el comando sale con código `0` (`healthy`); ante cualquier error `4xx`/`5xx` o fallo de conexión, lanza una excepción (`exit ≠ 0`).

### Healthcheck de `catalogo-frontend` (Nginx Alpine)

El servicio `catalogo-frontend` verifica su propia salud utilizando `wget` (incluido en BusyBox dentro de la imagen `nginxinc/nginx-unprivileged:1.27-alpine`, sin instalar paquetes adicionales):

```yaml
healthcheck:
  test:
    - CMD-SHELL
    - wget -q -O /dev/null http://127.0.0.1:8080/healthz || exit 1
  interval: 5s
  timeout: 3s
  retries: 5
  start_period: 5s
```

- **Puerto interno `8080`:** El healthcheck corre dentro del contenedor, por lo que `127.0.0.1:8080` apunta al proceso de Nginx y no al puerto `3000` publicado en el host. `-O /dev/null` descarta la salida para no escribir archivos en disco.
- **Por qué consulta `/healthz` y no `/api/health`:** El healthcheck del frontend debe medir únicamente si **Nginx está respondiendo**. Si consultara `/api/health`, una caída de la API marcaría como `unhealthy` al frontend, acoplando la salud de una capa a la de otra.

### Orden de arranque y dependencias (`depends_on` con `condition: service_healthy`)

Tanto `catalogo-api` como `catalogo-frontend` declaran su dependencia esperando que la capa inferior alcance el estado `healthy`, pero **los dos casos no son equivalentes**:

- **`catalogo-api` respecto de `catalogo-db`:** La API necesita **conectarse efectivamente a MongoDB y ejecutar la siembra del catálogo** al arrancar. La forma corta (`depends_on: [catalogo-db]`) solo espera a que el contenedor haya iniciado, no a que `mongod` termine de arrancar y acepte conexiones (lo que puede derivar en `ServerSelectionTimeoutError`). Con `condition: service_healthy`, Compose espera al `ping` exitoso de `mongosh`. Además, la API implementa reintentos de conexión (~30 s) en su propio código: esto no es redundante, porque **en Kubernetes no existe `depends_on`** y ese reintento será lo único que evite fallos al desplegar en el clúster.
- **`catalogo-frontend` respecto de `catalogo-api`:** Aquí lo imprescindible es declarar **algún `depends_on`**. Sin él, el frontend arranca antes de que exista `catalogo-api` en el DNS interno de `catalogo-net`, Nginx no puede resolver el *upstream* y termina inmediatamente (`Exited (1)`) con `host not found in upstream`. Para evitar ese fallo a Nginx le bastaría la forma corta (solo necesita que el nombre DNS resuelva al iniciar); usar `condition: service_healthy` aporta además que el frontend arranque con la API ya operativa, evitando respuestas `502 Bad Gateway` en los primeros segundos.

### Verificación de persistencia tras bajar y volver a levantar el stack

```bash
$ docker compose down
 ✔ Container catalogo-frontend Removed
 ✔ Container catalogo-api      Removed
 ✔ Container catalogo-db       Removed
 ✔ Network catalogo-net        Removed
```
```bash
$ docker compose up -d
 ✔ Network catalogo-net        Created
 ✔ Container catalogo-db       Healthy
 ✔ Container catalogo-api      Healthy
 ✔ Container catalogo-frontend Started
```
```bash
$ curl -s localhost:3000/api/productos | head -c 200
  [{"sku":999666,"categoria":"Apple","creado_en":"2026-11-22T12:29:47.538000","descripcion":"Clásico de 4.7\" con Touch ID y cuerpo de aluminio.","imagen_hash":"7e23251b780a76a1","imagen_tipo":"image/j
```
```bash
$ curl -s http://localhost:3000/api/productos | python3 -c "import sys,json; print('Productos:', len(json.load(sys.stdin)))"
Productos: 13
```
```bash
$ docker compose down -v
 ✔ Container catalogo-frontend Removed
 ✔ Container catalogo-api      Removed
 ✔ Container catalogo-db       Removed
 ! Volume catalogo-db-data     Skipped Volume was not created by Docker Compose
 ✔ Network catalogo-net        Removed
```
```bash
$ docker volume ls | grep catalogo-db-data
local     catalogo-db-data
```
- **`docker compose down`:** Elimina los contenedores y la red `catalogo-net`, pero **conserva el volumen `catalogo-db-data`** y todos los datos almacenados (se recuperan los 13 productos al volver a ejecutar `up -d`).
- **`docker compose down -v`:** Elimina además los volúmenes declarados del proyecto y borra los datos; solo debe utilizarse cuando se quiere reiniciar la base de datos desde cero a propósito. Aunque, por seguridad, `docker compose down -v` solo elimina volúmenes que fueron creados por Compose, y salta (`Skipped`) los volúmenes preexistentes para no destruir datos externos por accidente.

### Idempotencia de la siembra de datos

```bash
$ curl -s http://localhost:3000/api/productos | python3 -c "import sys,json; print('Antes:', len(json.load(sys.stdin)))"
Antes: 13
```
```bash
$ docker compose up -d --force-recreate catalogo-api
 ✔ Container catalogo-db  Healthy
 ✔ Container catalogo-api Started
```
```bash
$ curl -s http://localhost:3000/api/productos | python3 -c "import sys,json; print('Después:', len(json.load(sys.stdin)))"
Después: 13
```
- Al recrear `catalogo-api`, la aplicación vuelve a ejecutar la siembra inicial (`SEED_DEMO=true`) sobre la base ya poblada, y la cantidad de productos **no se duplica** (`13` antes y `13` después).
- Esto ocurre porque la siembra en `backend/main.py` utiliza `update_one(..., upsert=True)` por `sku` (con índice único) en lugar de un `insert` simple: condición indispensable para escalar `catalogo-api` a múltiples réplicas simultáneas en Kubernetes sin duplicar el catálogo.

### Revisión de seguridad y escaneo de vulnerabilidades (Trivy)

```bash
$ docker compose exec catalogo-api id
uid=1000(appuser) gid=1000(appuser)

$ docker compose exec catalogo-frontend id
uid=101(nginx) gid=101(nginx)

$ docker compose exec catalogo-db id
uid=0(root)
```
```bash
# Escaneo con Trivy 
$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image catalogo-api:v1

$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image catalogo-frontend:v1
```
```bash
# Fixed Version
$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --ignore-unfixed -q catalogo-api:v1 | grep -E "Total:|debian|python-pkg"

$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --ignore-unfixed -q catalogo-frontend:v1 | grep -E "Total:|alpine"
```

| Servicio / Imagen | Usuario (`id`) | Puertos (`docker compose ps`) | Sistema base (Total / con `Fixed Version`) | Dependencias Python (Total / con `Fixed Version`) | Observaciones |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `catalogo-frontend:v1` | `uid=101(nginx)` (no-root) | `0.0.0.0:3000->8080/tcp` | `alpine 3.21.3`: **217** (`3 critical, 65 high`) / **217** con parche | N/A (solo estáticos compilados; sin Node en runtime) | Único punto de entrada expuesto al host. Todas las CVE tienen parche actualizando la imagen base. |
| `catalogo-api:v1` | `uid=1000(appuser)` (no-root) | `0.0.0.0:8000->8000/tcp` | `debian 13.7`: **247** (`0 critical, 51 high`) / **30** con parche | **18** (`0 critical, 6 high`: `pymongo`, `starlette`, `pip`) / **18** con parche | Puerto `8000` publicado **solo para depuración**; en Kubernetes será un `Service` interno. |
| `catalogo-db` (`mongo:7`) | `uid=0(root)` en `exec` (entrypoint baja a `mongodb`) | `27017/tcp` (**no publicado**) | Imagen oficial externa | N/A | Alcanzable únicamente desde la red interna `catalogo-net`. |

- **Lectura del escaneo:** La columna determinante es `Fixed Version`. En `catalogo-frontend:v1`, el 100 % de los hallazgos (`217/217`) tiene parche y se corrige reconstruyendo sobre una imagen base más reciente. En `catalogo-api:v1`, las `18` vulnerabilidades de Python tienen parche actualizando librerías, mientras que en el sistema base Debian la mayoría (`217/247`) no tiene parche disponible hoy y se documenta para decidir (aceptar, mitigar o cambiar de base).

### Principio de mínimo privilegio (`cap_drop` / `cap_add`)

Se aplicó `cap_drop: [ALL]` a los tres servicios en `docker-compose.yml`. Las dos imágenes propias (`catalogo-api` y `catalogo-frontend`) continuaron funcionando sin necesitar ninguna capability; en `catalogo-db` (`mongo:7`) se agregaron únicamente las capabilities estrictamente necesarias, de a una, leyendo el error de `docker compose logs catalogo-db` en cada paso:

| Paso | `cap_add` aplicado en `catalogo-db` | Error observado en `docker compose logs catalogo-db` | Motivo |
| :---: | :--- | :--- | :--- |
| 1 | *(ninguna — solo `cap_drop: [ALL]`)* | `find: '/data/db/.mongodb': Permission denied` | El entrypoint corre como `root` y necesita recorrer subdirectorios de `/data/db` propiedad de `mongodb`. |
| 2 | `DAC_OVERRIDE` | `chown: changing ownership of '/data/db/.mongodb/mongosh/...': Operation not permitted` | `find` logró pasar, pero el entrypoint necesita ajustar el propietario de los archivos en `/data/db`. |
| 3 | `DAC_OVERRIDE`, `CHOWN` | `error: failed switching to 'mongodb': operation not permitted` | `chown` pasó, pero `gosu` necesita cambiar de grupo y usuario (`root` → `mongodb`). |
| 4 | `DAC_OVERRIDE`, `CHOWN`, `SETGID` | `error: failed switching to 'mongodb': operation not permitted` | Cambia el GID, pero todavía falta permiso para cambiar el UID del proceso. |
| 5 | `DAC_OVERRIDE`, `CHOWN`, `SETGID`, `SETUID` | **Sin errores — los 3 servicios alcanzan `(healthy)`** | Lista mínima completa para `mongo:7`. |

- **Por qué `catalogo-db` requiere capabilities y las otras dos capas no:**
  - `catalogo-api` (`uid=1000`, `appuser`, puerto `8000`) y `catalogo-frontend` (`uid=101`, `nginx`, puerto `8080`) fueron construidas para ejecutarse desde el inicio como usuarios **no-root** en puertos no privilegiados (`> 1024`), por lo que no requieren ninguna Linux Capability.
  - `catalogo-db` (`mongo:7`) arranca su `docker-entrypoint.sh` como `root` para inspeccionar y corregir la propiedad de `/data/db` y luego bajar privilegios al usuario `mongodb` mediante `gosu`, por lo que necesita exactamente `DAC_OVERRIDE`, `CHOWN`, `SETGID` y `SETUID`.

### Comparativa: `docker run` manuales vs. `docker-compose.yml`

| Aspecto | Despliegue manual (`docker run`) | Despliegue declarativo (`docker-compose.yml` + `.env`) |
| :--- | :--- | :--- |
| **Cantidad de comandos** | 5 comandos imperativos largos (`network create`, `volume create` y tres `docker run`) con ~14 flags en total. | Un único archivo versionado en Git levantado con `docker compose up -d`. |
| **Orden de arranque** | Dependía de la memoria del operador; invertir el orden hacía fallar a Nginx (`host not found in upstream`). | Declarado explícitamente con `depends_on` y `condition: service_healthy`. |
| **Verificación de salud** | Espera manual revisando `docker logs catalogo-db` hasta ver `"Waiting for connections"`. | `healthcheck` nativo en las tres capas sin instalar dependencias (`mongosh ping`, `urllib`, `wget /healthz`). |
| **Validación previa** | Ninguna: cualquier error de tipeo se descubría recién al fallar el contenedor. | Validación de sintaxis y resolución de variables antes de levantar con `docker compose config`. |
| **Variables y credenciales** | Escritas en texto plano en los comandos de consola y en el historial de shell. | Separadas en `.env` (ignorado en `.gitignore`) + `.env.example` versionado, interpoladas con `${VAR:?...}`. |
| **Seguridad (privilegios)** | Sin restricción de capabilities (cada contenedor heredaba las ~14 por defecto de Docker). | Endurecimiento declarativo con `cap_drop: [ALL]` en las tres capas y solo 4 `cap_add` mínimos en `catalogo-db`. |

- **Qué desapareció:** los comandos largos y repetitivos en memoria, las esperas manuales mirando logs, el riesgo de romper el orden de arranque y las contraseñas hardcodeadas en los comandos.
- **Qué apareció:** infraestructura declarativa versionada, validación previa (`config`), chequeos automáticos de salud (`healthcheck`), arranque encadenado (`service_healthy`), contrato de variables (`.env` / `.env.example`) y mínimo privilegio (`cap_drop` / `cap_add`).

## Etapa 5 — Primer despliegue en Kubernetes (Pods sueltos)

### Contexto del clúster (verificación previa)

```bash
$ kubectl config current-context
jflores

$ kubectl cluster-info
Kubernetes control plane is running at https://172.16.21.16:6443
CoreDNS is running at https://172.16.21.16:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

$ kubectl get nodes -o wide
NAME      STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION                       CONTAINER-RUNTIME
jflores   Ready    control-plane   40d   v1.37.1   172.16.21.16   <none>        Debian GNU/Linux 13 (trixie)   6.12.116+deb13-cloud-amd64 (amd64)   containerd://1.7.24
```

### 1. Prueba de `ImagePullBackOff`

Prueba deliberada omitiendo el usuario del registry (`catalogo-api:v1` en lugar de `jmfloress04/catalogo-api:v1`):

```bash
$ kubectl run prueba-imagepull --image=catalogo-api:v1
pod/prueba-imagepull created

$ kubectl get pods
NAME               READY   STATUS         RESTARTS   AGE
prueba-imagepull   0/1     ErrImagePull   0          39s

$ kubectl describe pod prueba-imagepull
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  65s                default-scheduler  Successfully assigned default/prueba-imagepull to jflores
  Normal   BackOff    28s (x2 over 58s)  kubelet            spec.containers{prueba-imagepull}: Back-off pulling image "catalogo-api:v1"
  Warning  Failed     28s (x2 over 58s)  kubelet            spec.containers{prueba-imagepull}: Error: ImagePullBackOff
  Normal   Pulling    14s (x3 over 60s)  kubelet            spec.containers{prueba-imagepull}: Pulling image "catalogo-api:v1"
  Warning  Failed     13s (x3 over 58s)  kubelet            Failed to pull image "catalogo-api:v1": failed to pull and unpack image "docker.io/library/catalogo-api:v1": failed to resolve reference "docker.io/library/catalogo-api:v1": pull access denied, repository does not exist or may require authorization
  Warning  Failed     13s (x3 over 58s)  kubelet            Error: ErrImagePull

$ kubectl logs prueba-imagepull
Error from server (BadRequest): container "prueba-imagepull" in pod "prueba-imagepull" is waiting to start: trying and failing to pull image

$ kubectl delete pod prueba-imagepull
pod "prueba-imagepull" deleted from default namespace
```

- **Imagen que intentó descargar el `kubelet`:** `docker.io/library/catalogo-api:v1` (el espacio oficial de Docker Hub, al omitir el usuario `jmfloress04/`).
- **Por qué `kubectl logs` no devuelve nada:** porque el contenedor nunca llegó a crearse ni arrancar; en `ImagePullBackOff` la causa siempre está en los `Events` de `kubectl describe`.

### 2. Creación del namespace `catalogo-app` y configuración del contexto

```bash
$ kubectl config current-context
jflores

$ kubectl create namespace catalogo-app
namespace/catalogo-app created

$ kubectl config set-context --current --namespace=catalogo-app
Context "jflores" modified.

$ kubectl get namespaces
NAME                 STATUS   AGE
catalogo-app         Active   3m12s
default              Active   40d
kube-flannel         Active   40d
kube-node-lease      Active   40d
kube-public          Active   40d
kube-system          Active   40d
local-path-storage   Active   29d
metallb-system       Active   40d
nginx-ingress        Active   40d

$ kubectl config view --minify | grep namespace:
    namespace: catalogo-app
```

Verificar primero `kubectl config current-context` confirma que estamos operando sobre el clúster correcto (`jflores`) antes de aplicar cambios. Al crear el namespace `catalogo-app` y fijarlo como namespace por defecto del contexto, todos los comandos posteriores de `kubectl` apuntan automáticamente a `catalogo-app` en lugar de `default`, evitando crear o buscar Pods en el namespace equivocado.

### 3. Generación del primer manifiesto (`k8s/pod-catalogo-db.yaml`) con `--dry-run=client`

```bash
$ mkdir -p k8s
$ kubectl run catalogo-db --image=mongo:7 --dry-run=client -o yaml > k8s/pod-catalogo-db.yaml
$ cat k8s/pod-catalogo-db.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: catalogo-db
  name: catalogo-db
spec:
  containers:
  - image: mongo:7
    name: catalogo-db
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Luego se editó `k8s/pod-catalogo-db.yaml` limpiando los campos vacíos (`resources: {}` y `status: {}`) e incorporando el namespace `catalogo-app`, el puerto `27017` y las variables `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`.

**Conclusión:** Generar el esqueleto YAML con `--dry-run=client -o yaml` y luego editarlo evita memorizar toda la estructura y reduce errores de sintaxis; además no aplica nada todavía contra el clúster, permitiendo revisar y versionar el manifiesto antes de desplegarlo.

### 4. Despliegue de las tres capas como Pods sueltos y registro de IPs

Se crearon `k8s/pod-catalogo-api.yaml` (incluyendo `ROOT_PATH=/api`) y `k8s/pod-catalogo-frontend.yaml` con las variables de la semana 3, y se aplicaron los tres manifiestos en orden (`db` → `api` → `frontend`):

```bash
$ kubectl apply -f k8s/pod-catalogo-db.yaml
pod/catalogo-db created

$ kubectl apply -f k8s/pod-catalogo-api.yaml
pod/catalogo-api created

$ kubectl apply -f k8s/pod-catalogo-frontend.yaml
pod/catalogo-frontend created

$ kubectl get pods -w
NAME                READY   STATUS              RESTARTS      AGE
catalogo-api        0/1     ContainerCreating   0             26s
catalogo-db         1/1     Running             0             97s
catalogo-frontend   0/1     ContainerCreating   0             8s
catalogo-api        1/1     Running             0             68s
catalogo-frontend   1/1     Running             0             63s
catalogo-frontend   0/1     Error               0             71s
catalogo-frontend   0/1     CrashLoopBackOff    1 (15s ago)   87s
catalogo-api        0/1     Error               0             2m7s
catalogo-api        1/1     Running             1 (2s ago)    2m8s
catalogo-api        0/1     Error               1 (61s ago)   3m7s
catalogo-api        0/1     CrashLoopBackOff    1 (11s ago)   3m17s
catalogo-api        1/1     Running             2 (12s ago)   3m18s

$ kubectl get pods -o wide
NAME                READY   STATUS    RESTARTS      AGE     IP            NODE      NOMINATED NODE   READINESS GATES
catalogo-api        1/1     Running   2 (21s ago)   3m27s   10.244.0.25   jflores   <none>           <none>
catalogo-db         1/1     Running   0             4m38s   10.244.0.24   jflores   <none>           <none>
catalogo-frontend   0/1     Error     4 (75s ago)   3m9s    10.244.0.26   jflores   <none>           <none>
```

**Direcciones IP asignadas a cada Pod:**

| Pod | IP asignada | Nodo | Estado observado |
| :--- | :---: | :---: | :--- |
| `catalogo-db` | `10.244.0.24` | `jflores` | `1/1 Running` (`0` reinicios) |
| `catalogo-api` | `10.244.0.25` | `jflores` | `Error` / `CrashLoopBackOff` (`2` reinicios; reintenta ~60 s antes de fallar) |
| `catalogo-frontend` | `10.244.0.26` | `jflores` | `Error` / `CrashLoopBackOff` (`4` reinicios; falla al instante) |

**Conclusión:** Aunque los tres Pods están desplegados en el mismo namespace (`catalogo-app`) y en el mismo nodo (`jflores`), solo `catalogo-db` se mantiene estable porque no depende de resolver ningún otro nombre. Tanto `catalogo-api` (que busca `catalogo-db`) como `catalogo-frontend` (que busca `catalogo-api`) entran en `CrashLoopBackOff` porque en Kubernetes un Pod suelto no registra ningún nombre DNS; además, a diferencia de Compose, en Kubernetes no existe `depends_on` y `Running` no equivale a `Ready` sin una `readinessProbe` (y en MongoDB el proceso puede aceptar conexiones TCP incluso antes de terminar de crear el usuario administrador, por lo que `depends_on` + `condition: service_healthy` se reemplaza con *probes* y reintentos en la aplicación).

### 5. Diagnóstico del `CrashLoopBackOff` de `catalogo-api`

```bash
$ kubectl describe pod catalogo-api
$ kubectl logs catalogo-api --previous
```

*Nota: al consultar `kubectl logs catalogo-api --previous` mientras el contenedor actual ya estaba en `Terminated` esperando backoff, `containerd` devolvió `unable to retrieve container logs` sobre la instancia más antigua; los logs del fallo completo se obtuvieron directamente con `kubectl logs catalogo-api`.*

**Conclusión:** El error informado por `pymongo` es **`ServerSelectionTimeoutError`** (`[Errno -2] Name or service not known`) y **NO** `Authentication failed`. Esto ocurre porque `DB_HOST=catalogo-db` ni siquiera pudo resolverse en DNS: en Kubernetes un Pod suelto no provee un nombre resoluble a ningún otro Pod (esa es la función de un **`Service`**, que todavía no existe). Al fallar en la resolución de nombres antes de abrir la conexión TCP, la API nunca alcanzó a MongoDB y por eso jamás llegó a la instancia de autenticación donde aparecería `Authentication failed`.

### 6. Conexión temporal por IP y por qué la IP de un Pod es descartable 

Al reemplazar temporalmente `DB_HOST` en `k8s/pod-catalogo-api.yaml` por la dirección IP original del Pod de MongoDB (`10.244.0.24`) y recrear `catalogo-api`, la API logró conectarse a la base de datos y alcanzó el estado `1/1 Running` sin reinicios.

A continuación se eliminó el Pod `catalogo-db` y se volvió a crear con el mismo manifiesto `k8s/pod-catalogo-db.yaml`. Kubernetes le asignó una dirección IP diferente (`<IP_NUEVA>` en lugar de `10.244.0.24`), provocando que `catalogo-api` volviera a fallar al seguir apuntando a una dirección inexistente.

**Conclusión:** La dirección IP de un Pod es efímera y descartable: cada vez que un Pod se elimina, se reprograma o se recrea, recibe una IP nueva del clúster, por lo que nunca sirve como punto de configuración. En ese hecho reside la razón de existir de los **`Services`** (Semana 6), que proveen una IP virtual estable y un nombre DNS fijo (`catalogo-db`) por delante de Pods cuyas IPs cambian constantemente.

*Nota de estado final: tras completar las pruebas de la Consigna 8, se restituyó `DB_HOST: "catalogo-db"` en `k8s/pod-catalogo-api.yaml`; el manifiesto queda deliberadamente roto hasta que se cree el `Service` correspondiente en la siguiente entrega.*

### 7. Tercer `host not found in upstream` en `catalogo-frontend`

Al desplegar `k8s/pod-catalogo-frontend.yaml`, Nginx vuelve a fallar al arrancar con el mensaje `host not found in upstream "catalogo-api:8000"`, por tercera vez a lo largo del curso:

| Etapa | Mensaje de error en Nginx | Causa raíz | Cómo se resuelve |
| :---: | :--- | :--- | :--- |
| **Semana 3** (`docker run`) | `host not found in upstream "catalogo-api:8000"` | **Faltaba la red custom** (`catalogo-net`) con DNS interno embebido, o la API aún no había sido creada. | Crear la red `bridge` custom `catalogo-net` y levantar `catalogo-api` antes del frontend. |
| **Semana 4** (`docker compose`) | `host not found in upstream "catalogo-api:8000"` | **Condición de carrera al arrancar:** sin `depends_on`, Compose iniciaba Nginx antes de que existiera `catalogo-api`. | Declarar `depends_on` (con `condition: service_healthy`) en `docker-compose.yml`. |
| **Semana 5** (Kubernetes — Pods sueltos) | `host not found in upstream "catalogo-api:8000"` | **Falta el `Service`:** aunque `catalogo-api` exista y esté corriendo, un Pod suelto no registra ningún nombre DNS en CoreDNS. | Crear un objeto **`Service`** llamado `catalogo-api` (Semana 6). |

**Conclusión:** Aunque el mensaje de error de Nginx es idéntico al de la tercera semana, la causa arquitectónica es distinta: entonces faltaba la red de Docker con DNS interno, mientras que en Kubernetes falta el **`Service`** que exponga el nombre `catalogo-api` dentro de CoreDNS.

### 8. Acceso temporal sin `Service` mediante `kubectl port-forward`

Con `catalogo-api` en `Running` (apuntando momentáneamente a la IP actual de `catalogo-db`), se abrió un túnel local con `kubectl port-forward pod/catalogo-api 8000:8000` y se probaron los endpoints:

```bash
$ curl -s localhost:8000/health
{"status":"ok","db":"ok","version":"v1"}

$ curl -s localhost:8000/productos | head -c 250
[{"sku":999666,"categoria":"Apple","creado_en":"2026-11-23T06:09:47.862000","descripcion":"Clásico de 4.7\" con Touch ID y cuerpo de aluminio.","imagen_hash":"7e23251b780a76a1","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 6 (Gold, 32 GB)"...
```

Al acceder a `http://localhost:8000/docs`, Swagger UI devuelve el HTML inicial (`200 OK`) pero **no logra cargar la definición de la API** porque intenta descargar `/api/openapi.json` y recibe **`404 Not Found`**.

**Conclusión:** El `404` en `/docs` (`/api/openapi.json`) **no es un error**, sino la consecuencia directa de haber configurado `ROOT_PATH=/api` desde la Semana 3 para funcionar detrás del proxy reverso Nginx: al entrar directo al Pod por `port-forward` no existe ningún proxy delante que maneje el prefijo `/api`. Además, **`kubectl port-forward` es únicamente una herramienta de depuración temporal y no un mecanismo de publicación**: escucha solo en `127.0.0.1` local, termina al cerrar la terminal y se interrumpe si el Pod se reinicia porque apunta a un Pod concreto y no a un `Service`.

### 9. Mortalidad de los Pods sueltos

Al eliminar el Pod `catalogo-api` y volver a listar los Pods del namespace `catalogo-app`, se comprobó que `catalogo-api` desapareció y Kubernetes no creó ningún reemplazo.

**Conclusión:** En Kubernetes existe autorreparación a nivel de **contenedor** (el `kubelet` reinicia un proceso caído gracias a `restartPolicy: Always`), pero **un Pod suelto es mortal y nadie lo vigila**: si se elimina el Pod entero o cae el nodo donde corría, desaparece y no vuelve. Para garantizar una cantidad deseada de réplicas y autorreparación a nivel de Pod se requieren controladores como **`ReplicaSet` y `Deployment`**.

### 10. Cheatsheet propio de `kubectl` (Consigna 10)

Los diez comandos de `kubectl` efectivamente utilizados durante esta entrega:

| # | Comando | Qué hace |
| :---: | :--- | :--- |
| 1 | `kubectl config current-context` | Muestra sobre qué clúster/contexto estoy operando antes de aplicar cambios. |
| 2 | `kubectl create namespace catalogo-app` | Crea el espacio de nombres lógico `catalogo-app` dentro del clúster. |
| 3 | `kubectl config set-context --current --namespace=catalogo-app` | Configura `catalogo-app` como namespace por defecto del contexto activo para no repetir `-n`. |
| 4 | `kubectl run catalogo-db --image=mongo:7 --dry-run=client -o yaml > k8s/pod-catalogo-db.yaml` | Genera el manifiesto YAML base de un Pod localmente sin crearlo todavía en el clúster. |
| 5 | `kubectl apply -f k8s/pod-catalogo-db.yaml` | Aplica un manifiesto de forma declarativa e idempotente contra la API de Kubernetes. |
| 6 | `kubectl get pods -o wide` | Lista los Pods del namespace activo mostrando estado, reinicios, dirección IP y nodo asignado. |
| 7 | `kubectl get pods -w` | Observa en tiempo real las transiciones de estado de los Pods (`ContainerCreating`, `Running`, `Error`, `CrashLoopBackOff`). |
| 8 | `kubectl describe pod catalogo-api` | Muestra el detalle completo de un Pod y su sección `Events` (fundamental para diagnosticar `ImagePullBackOff` o reinicios). |
| 9 | `kubectl logs catalogo-api` *(y `--previous`)* | Muestra la salida estándar de la aplicación dentro del Pod (con `--previous` consulta la encarnación anterior tras un reinicio). |
| 10 | `kubectl port-forward pod/catalogo-api 8000:8000` | Abre un túnel temporal local hacia un puerto de un Pod específico únicamente para depuración. |

### 11. Comparación: `docker-compose.yml` vs. manifiestos `k8s/`

| Aspecto | `docker-compose.yml` (Semana 4) | Manifiestos `k8s/pod-*.yaml` (Semana 5) | Qué ocurrió |
| :--- | :--- | :--- | :--- |
| **Imágenes OCI** | `build:` local + `image:` en caché del Docker Engine | `image: jmfloress04/catalogo-*:v1` descargadas desde Docker Hub | **Se conservó** la misma imagen sin cambios; cambió la distribución vía registry. |
| **Variables de entorno** | Bloque `environment:` con interpolación desde `.env` | Lista `env:` (`name`/`value` como strings) dentro de cada manifiesto | **Se conservó** el contrato (`DB_*`, `ROOT_PATH=/api`, `API_*`), pero los secretos volvieron inline (**faltan `ConfigMap` y `Secret`**). |
| **Red y resolución DNS** | Red custom `catalogo-net` con DNS interno automático por nombre | Pods sueltos con IP efímera y **sin** registro DNS en CoreDNS | **Desapareció** la resolución por nombre (**falta `Service`**). |
| **Orden de arranque y salud** | `depends_on` (`condition: service_healthy`) + `healthcheck` | **No existe `depends_on`**; `Running` no equivale a `Ready` | **Desapareció** `depends_on` (**faltan `readinessProbe` / `livenessProbe`** + reintentos en la app). |
| **Exposición de puertos** | `ports: ["3000:8080"]` publicaba directo al host | `containerPort` es solo informativo; acceso solo con `port-forward` | **Desapareció** la publicación directa (**faltan `Service` e `Ingress`**). |
| **Persistencia de datos** | Volumen nombrado `catalogo-db-data` montado en `/data/db` | Sin volumen declarado (al recrear `catalogo-db` se pierden los datos) | **Falta almacenamiento persistente (`PV` / `PVC`)**. |
| **Autorreparación y réplicas** | `restart` en un único host | `kubelet` reinicia contenedores, pero un Pod borrado no se recrea | **Faltan `Deployment`, `ReplicaSet` / `StatefulSet`**. |
| **Seguridad (`cap_drop`)** | `cap_drop: [ALL]` + 4 `cap_add` mínimas en `mongo:7` | Sin restricciones de capabilities declaradas todavía | **Falta configurar `securityContext`**. |

**Conclusión:** Esta entrega conserva intactas las imágenes y el contrato de variables de entorno de la Semana 4, pero expone de forma deliberada todo lo que falta al migrar únicamente a Pods sueltos: nombres DNS estables (`Service`), autorreparación a nivel de Pod (`Deployment`/`StatefulSet`), sondas de salud (`probes`), configuración desacoplada (`ConfigMap`/`Secret`), persistencia (`PV`/`PVC`) y exposición externa (`Ingress`), recursos que se incorporarán progresivamente en las semanas 6 a 8.