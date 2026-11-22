## Arquitectura

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

## Medición y Optimización de Imágenes

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

## Redes y Almacenamiento

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