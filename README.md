# Catálogo App — Variante C

Catálogo de productos con imágenes (**React + Vite** servido por **Nginx unprivileged** · **Python + FastAPI** · **MongoDB 7**). Proyecto integrador de *Contenedores, Docker y Orquestación con Kubernetes* (FACET – UNT).

### Documentación y artefactos de la entrega final

| Artefacto | Ubicación | Descripción |
| :--- | :--- | :--- |
| Dockerfiles multietapa | `backend/Dockerfile`, `frontend/Dockerfile` | Imágenes sin herramientas de compilación, no-root y configuración en tiempo de ejecución. |
| Docker Compose | `docker-compose.yml`, `.env.example` | Las tres capas en una sola máquina con red `catalogo-net`, volumen persistente y healthchecks. |
| Manifiestos de Kubernetes | `k8s/` | Los **12 objetos** del despliegue en archivos individuales (`3` Deployments, `3` Services, `2` ConfigMaps, `1` Secret ejemplo, `1` PVC, `1` Ingress, `1` HPA). |
| Chart de Helm | `catalogo-app/` | `Chart.yaml`, `values.yaml` y las **12 plantillas** parametrizadas en `templates/`. |
| Documento de arquitectura | [ARQUITECTURA.md](ARQUITECTURA.md) | Respuestas a las preguntas obligatorias, pregunta propia de la Variante C, dificultades reales de MongoDB y pendientes para producción. |
| Informe de funcionamiento | [INFORME.md](INFORME.md) | Demostración con registros de comandos reales (Ingress, HPA `-w`, `helm history`, doble `helm template` y prueba *stateless* a 3 réplicas). |

## Índice

- **[Cómo levantar el proyecto](#cómo-levantar-el-proyecto)**
  - [Camino 1 — Con Docker Compose (en una sola máquina)](#camino-1--con-docker-compose-en-una-sola-máquina)
  - [Camino 2 — Con Helm (sobre el clúster Kubernetes)](#camino-2--con-helm-sobre-el-clúster-kubernetes)
  - [Credenciales de laboratorio y manifiestos individuales (`k8s/`)](#credenciales-de-laboratorio-y-manifiestos-individuales-k8s)
- **[Etapa 1 — Arquitectura](#etapa-1--arquitectura)**
- **[Etapa 2 — Medición y Optimización de Imágenes](#etapa-2--medición-y-optimización-de-imágenes)**
  - [Qué aportó cada decisión](#qué-aportó-cada-decisión)
  - [Análisis de capas (`docker history`)](#análisis-de-capas-docker-history)
  - [Efecto del cache de construcción](#efecto-del-cache-de-construcción)
- **[Etapa 3 — Redes y Almacenamiento](#etapa-3--redes-y-almacenamiento)**
  - [Red y volumen](#red-y-volumen)
  - [Base de datos (`catalogo-db`)](#base-de-datos-catalogo-db)
  - [API (`catalogo-api`) y resolución DNS interna](#api-catalogo-api-y-resolución-dns-interna)
  - [Diagnóstico de fallas: resolución de nombre vs. autenticación](#diagnóstico-de-fallas-resolución-de-nombre-vs-autenticación)
  - [Fallo del frontend fuera de la red (`catalogo-frontend`)](#fallo-del-frontend-fuera-de-la-red-catalogo-frontend)
  - [Frontend conectado a `catalogo-net` y verificación del reverse proxy](#frontend-conectado-a-catalogo-net-y-verificación-del-reverse-proxy)
  - [Verificación del proxy y efecto de `ROOT_PATH` en FastAPI](#verificación-del-proxy-y-efecto-de-root_path-en-fastapi)
  - [Persistencia de datos en MongoDB (`catalogo-db-data`)](#persistencia-de-datos-en-mongodb-catalogo-db-data)
  - [Orden de arranque manual](#orden-de-arranque-manual)
- **[Etapa 4 — Docker Compose y Seguridad Básica](#etapa-4--docker-compose-y-seguridad-básica)**
  - [Healthcheck de `catalogo-db` (MongoDB 7)](#healthcheck-de-catalogo-db-mongodb-7)
  - [Healthcheck de `catalogo-api` (FastAPI)](#healthcheck-de-catalogo-api-fastapi)
  - [Healthcheck de `catalogo-frontend` (Nginx Alpine)](#healthcheck-de-catalogo-frontend-nginx-alpine)
  - [Orden de arranque y dependencias (`depends_on` con `condition: service_healthy`)](#orden-de-arranque-y-dependencias-depends_on-con-condition-service_healthy)
  - [Verificación de persistencia tras bajar y volver a levantar el stack](#verificación-de-persistencia-tras-bajar-y-volver-a-levantar-el-stack)
  - [Idempotencia de la siembra de datos](#idempotencia-de-la-siembra-de-datos)
  - [Revisión de seguridad y escaneo de vulnerabilidades (Trivy)](#revisión-de-seguridad-y-escaneo-de-vulnerabilidades-trivy)
  - [Principio de mínimo privilegio (`cap_drop` / `cap_add`)](#principio-de-mínimo-privilegio-cap_drop--cap_add)
  - [Comparativa: `docker run` manuales vs. `docker-compose.yml`](#comparativa-docker-run-manuales-vs-docker-composeyml)
- **[Etapa 5 — Primer despliegue en Kubernetes (Pods sueltos)](#etapa-5--primer-despliegue-en-kubernetes-pods-sueltos)**
  - [Contexto del clúster (verificación previa)](#contexto-del-clúster-verificación-previa)
  - [1. Prueba de `ImagePullBackOff`](#1-prueba-de-imagepullbackoff)
  - [2. Creación del namespace `catalogo-app` y configuración del contexto](#2-creación-del-namespace-catalogo-app-y-configuración-del-contexto)
  - [3. Generación del primer manifiesto (`k8s/pod-catalogo-db.yaml`) con `--dry-run=client`](#3-generación-del-primer-manifiesto-k8spod-catalogo-dbyaml-con---dry-runclient)
  - [4. Despliegue de las tres capas como Pods sueltos y registro de IPs](#4-despliegue-de-las-tres-capas-como-pods-sueltos-y-registro-de-ips)
  - [5. Diagnóstico del `CrashLoopBackOff` de `catalogo-api`](#5-diagnóstico-del-crashloopbackoff-de-catalogo-api)
  - [6. Conexión temporal por IP y por qué la IP de un Pod es descartable](#6-conexión-temporal-por-ip-y-por-qué-la-ip-de-un-pod-es-descartable)
  - [7. Tercer `host not found in upstream` en `catalogo-frontend`](#7-tercer-host-not-found-in-upstream-en-catalogo-frontend)
  - [8. Acceso temporal sin `Service` mediante `kubectl port-forward`](#8-acceso-temporal-sin-service-mediante-kubectl-port-forward)
  - [9. Mortalidad de los Pods sueltos](#9-mortalidad-de-los-pods-sueltos)
  - [10. Cheatsheet propio de `kubectl` (Consigna 10)](#10-cheatsheet-propio-de-kubectl-consigna-10)
  - [11. Comparación: `docker-compose.yml` vs. manifiestos `k8s/`](#11-comparación-docker-composeyml-vs-manifiestos-k8s)
- **[Etapa 6 — Deployments, Services, Rolling Updates y Réplicas sin Estado](#etapa-6--deployments-services-rolling-updates-y-réplicas-sin-estado)**
  - [Orden de aplicación (Services antes que Deployments)](#orden-de-aplicación-services-antes-que-deployments)
  - [Verificación de selectors mediante Endpoints](#verificación-de-selectors-mediante-endpoints)
  - [Comprobación de autorreparación](#comprobación-de-autorreparación)
  - [Actualización progresiva (`RollingUpdate`) y `rollback`](#actualización-progresiva-rollingupdate-y-rollback)
  - [Escalado a 3 réplicas y por qué la API debe ser sin estado (*stateless*)](#escalado-a-3-réplicas-y-por-qué-la-api-debe-ser-sin-estado-stateless)
  - [Limitación conocida: `catalogo-db` sin persistencia](#limitación-conocida-catalogo-db-sin-persistencia)
- **[Etapa 7 — ConfigMaps, Secrets, Almacenamiento e Ingress](#etapa-7--configmaps-secrets-almacenamiento-e-ingress)**
  - [Secret `catalogo-db-secret` (Base64 no es cifrado)](#secret-catalogo-db-secret-base64-no-es-cifrado)
  - [Conexión a MongoDB: piezas sueltas vs. `MONGO_URI` completa](#conexión-a-mongodb-piezas-sueltas-vs-mongo_uri-completa)
  - [Persistencia de MongoDB: `catalogo-db-pvc`](#persistencia-de-mongodb-catalogo-db-pvc)
  - [Verificación de persistencia y cierre de la limitación](#verificación-de-persistencia-y-cierre-de-la-limitación)
  - [Exposición HTTP con Ingress: `catalogo-app-ingress`](#exposición-http-con-ingress-catalogo-app-ingress)
  - [Documentación de la API detrás del Ingress](#documentación-de-la-api-detrás-del-ingress)
  - [Inventario de los 11 objetos actuales del proyecto](#inventario-de-los-11-objetos-actuales-del-proyecto)
- **[Etapa 8 — Escalado Automático (HPA) y Empaquetado con Helm](#etapa-8--escalado-automático-hpa-y-empaquetado-con-helm)**
  - [1. Declaración de `resources.requests` y `limits` en `catalogo-api`](#1-declaración-de-resourcesrequests-y-limits-en-catalogo-api)
  - [2. Objeto N° 12: Creación del `HorizontalPodAutoscaler` (`catalogo-api-hpa`)](#2-objeto-n-12-creación-del-horizontalpodautoscaler-catalogo-api-hpa)
  - [3. Demostración bajo carga real (`2 → 4 → 6 → 3 → 2`)](#3-demostración-bajo-carga-real-2--4--6--3--2)
  - [4 y 5. Chart de Helm `catalogo-app/` (12 objetos exactos) y parametrización efectiva](#4-y-5-chart-de-helm-catalogo-app-12-objetos-exactos-y-parametrización-efectiva)
  - [6. Instalación limpia de la Release con `helm install --create-namespace`](#6-instalación-limpia-de-la-release-con-helm-install---create-namespace)
  - [7. Ciclo completo de Release: `helm upgrade` (`v2`), `helm history` y `helm rollback` (`v1`)](#7-ciclo-completo-de-release-helm-upgrade-v2-helm-history-y-helm-rollback-v1)

---

## Cómo levantar el proyecto

### Camino 1 — Con Docker Compose (en una sola máquina)

**Requisitos:** Docker Engine y Docker Compose v2 (`docker compose`).

1. **Crear el archivo `.env` local** a partir de la plantilla (`.env` está excluido en `.gitignore` para no versionar contraseñas):
   ```bash
   cp .env.example .env
   ```
   Editar `.env` definiendo las credenciales de laboratorio (asegurando que `MONGO_INITDB_ROOT_USERNAME`/`PASSWORD` coincidan con `DB_USER`/`DB_PASSWORD`).

2. **Construir las imágenes y levantar las tres capas:**
   ```bash
   docker compose up -d --build
   ```
   Compose crea la red `catalogo-net`, el volumen persistente `catalogo-db-data` y respeta el orden de arranque mediante healthchecks (`catalogo-db` → `catalogo-api` → `catalogo-frontend`).

3. **Verificar estado y acceder a la aplicación:**
   ```bash
   docker compose ps
   ```
   - **Aplicación web (Frontend + proxy):** `http://localhost:3000`
   - **Salud de la API (a través del proxy):** `http://localhost:3000/api/health`
   - **Documentación interactiva (Swagger UI):** `http://localhost:3000/api/docs`

4. **Detener el entorno:**
   ```bash
   # Detiene y elimina contenedores conservando los datos en catalogo-db-data:
   docker compose down

   # Para borrar también el volumen de MongoDB y reiniciar desde cero:
   docker compose down -v
   ```

### Camino 2 — Con Helm (sobre el clúster Kubernetes)

**Requisitos:** clúster Kubernetes con `kubectl` configurado, `Helm 3`, `ingress-nginx` (Ingress Controller), aprovisionador de almacenamiento por defecto (`StorageClass`) y `metrics-server` (para el HPA).

1. **Validar el Chart y las plantillas antes de instalar:**
   ```bash
   helm lint ./catalogo-app
   ```

2. **Instalar los 12 objetos en un único comando** (pasando credenciales de laboratorio por `--set` o mediante un archivo local `catalogo-app/values-secret.yaml`, ignorado en `.gitignore`):
   ```bash
   # Opción A: pasando credenciales de laboratorio por --set
   helm install catalogo-app ./catalogo-app \
     -n catalogo-app --create-namespace \
     --set db.auth.username=catalogo_lab_user \
     --set db.auth.password=catalogo_lab_pass

   # Opción B: usando un archivo local de overrides (ignorado por Git)
   helm install catalogo-app ./catalogo-app \
     -n catalogo-app --create-namespace \
     -f catalogo-app/values-secret.yaml
   ```

3. **Verificar el despliegue completo y configurar resolución local del Ingress:**
   ```bash
   kubectl get all,cm,secret,pvc,ingress,hpa -n catalogo-app
   ```
   Tomar la IP listada en la columna `ADDRESS` de `catalogo-app-ingress` y agregarla en `/etc/hosts` apuntando al host configurado en `values.yaml`:
   ```text
   <INGRESS_ADDRESS> catalogo.local
   ```
   Luego abrir:
   - **Aplicación web:** `http://catalogo.local`
   - **API detrás del Ingress:** `http://catalogo.local/api/productos`
   - **Swagger UI:** `http://catalogo.local/api/docs`

4. **Ciclo de vida de la release (actualización, historial, reversión y desinstalación):**
   ```bash
   # Actualizar la API a v2 (repetir los valores de credenciales o usar --reuse-values):
   helm upgrade catalogo-app ./catalogo-app -n catalogo-app \
     --reuse-values --set api.image.tag=v2

   # Consultar revisiones y revertir a la revisión 1:
   helm history catalogo-app -n catalogo-app
   helm rollback catalogo-app 1 -n catalogo-app

   # Desinstalar la release:
   helm uninstall catalogo-app -n catalogo-app
   ```

### Credenciales de laboratorio y manifiestos individuales (`k8s/`)

- **Sin credenciales reales en el repositorio:** tanto `.env.example`, `k8s/secret-catalogo-db.yaml.example` como `catalogo-app/values.yaml` contienen únicamente valores de ejemplo / placeholders de laboratorio. Los archivos con credenciales locales (`.env`, `k8s/secret-catalogo-db.yaml`, `catalogo-app/values-secret.yaml`) están excluidos explícitamente en `.gitignore`. La estrategia de gestión segura de secretos en producción se detalla en [ARQUITECTURA.md](ARQUITECTURA.md#4-qué-quedaría-pendiente-para-un-entorno-de-producción-real).
- **Si se desea aplicar los 12 manifiestos sueltos de `k8s/` sin Helm:**
  1. Copiar `k8s/secret-catalogo-db.yaml.example` a `k8s/secret-catalogo-db.yaml` y completar credenciales de laboratorio.
  2. Crear el namespace (`kubectl create namespace catalogo-app`) y aplicar respetando dependencias (`Secret`, `ConfigMaps`, `PVC` y `Services` antes que los `Deployments`, `HPA` e `Ingress`).

---

<!-- Bitácora completa del curso: Etapas 1 a 8 -->

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

El producto cargado desde el navegador en el paso 6 continuó disponible **con su imagen y con su identificador original**, porque los bytes de la imagen se almacenan dentro del documento en MongoDB (en el volumen `catalogo-db-data`) y no en el disco efímero del contenedor `catalogo-api`. Esto mantiene a la API sin estado (*stateless*) y permite escalarla a múltiples réplicas en Kubernetes (Etapa 6).

### Orden de arranque manual

Para levantar `catalogo-app` con `docker run` debe respetarse estrictamente este orden:

**`red (catalogo-net)` → `volumen (catalogo-db-data)` → `base (catalogo-db)` → `API (catalogo-api)` → `frontend (catalogo-frontend)`**

**Qué falla al invertir cada paso:**
- **Contenedor antes que la red:** `docker run --network catalogo-net` falla porque la red no existe (o, si se omite `--network`, el contenedor cae en `bridge` por defecto y no resuelve nombres por DNS).
- **Base de datos sin el volumen nombrado:** los datos se guardan en la capa de escritura efímera del contenedor y se pierden definitivamente con `docker rm`.
- **API antes que la base de datos:** la API no logra conectar con MongoDB al arrancar (reintentos fallidos en `esperar_mongo()` hasta agotar intentos y salir con error).
- **Frontend antes que la API:** Nginx resuelve el *upstream* `catalogo-api` una única vez al iniciar; si aún no existe en el DNS de `catalogo-net`, no reintenta y muere de inmediato (`Exited (1)`) con `nginx: [emerg] host not found in upstream "catalogo-api"`.

**Limitación actual (resuelta en la Etapa 4):**
Todo este orden de arranque, las variables de entorno, la red y el volumen residen hoy únicamente en la memoria del operador (o como comandos sueltos en este `README.md`) y no en ningún archivo ejecutable. En la Etapa 4, **Docker Compose** (`docker-compose.yml`) reemplaza esta secuencia manual por una definición declarativa reproducible con un solo comando.

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

Se crearon `k8s/pod-catalogo-api.yaml` (incluyendo `ROOT_PATH=/api`) y `k8s/pod-catalogo-frontend.yaml` con las variables de la etapa 3, y se aplicaron los tres manifiestos en orden (`db` → `api` → `frontend`):

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
```
```text
...
Events:
  Normal   Scheduled  8m27s                 default-scheduler  Successfully assigned catalogo-app/catalogo-api to jflores
  Normal   Pulled     108s (x4 over 6m20s)  kubelet            Container image "jmfloress04/catalogo-api:v1" already present on machine and can be accessed by the pod
  Normal   Created    107s (x5 over 7m21s)  kubelet            Container created
  Normal   Started    107s (x5 over 7m19s)  kubelet            Container started
  Warning  BackOff    47s (x4 over 5m20s)   kubelet            Back-off restarting failed container catalogo-api in pod catalogo-api_catalogo-app(...)
```
```bash
$ kubectl logs catalogo-api --previous
```
```text
unable to retrieve container logs for containerd://fa3cd58937b9b48bc149f5f316943d86647dbcceeef40b393a597ded46e7c9c0
```
```bash
$ kubectl logs catalogo-api
```
```text
INFO:     Started server process [1]
INFO:     Waiting for application startup.
[2026-11-23T05:36:33+0000] WARNING intento 1/10 de conexión falló: catalogo-db:27017: [Errno -2] Name or service not known ...
...
[2026-11-23T05:37:27+0000] WARNING intento 10/10 de conexión falló: catalogo-db:27017: [Errno -2] Name or service not known ...
pymongo.errors.ServerSelectionTimeoutError: catalogo-db:27017: [Errno -2] Name or service not known (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms), Timeout: 3.0s, Topology Description: <TopologyDescription id: 6b03d0ded19a167c7c7bb84f, topology_type: Unknown, servers: [<ServerDescription ('catalogo-db', 27017) server_type: Unknown, rtt: None, error=AutoReconnect('catalogo-db:27017: [Errno -2] Name or service not known ...')>]>
ERROR:    Application startup failed. Exiting.
```
*Nota: al consultar `kubectl logs catalogo-api --previous` mientras el contenedor actual ya estaba en `Terminated` esperando backoff, `containerd` devolvió `unable to retrieve container logs` sobre la instancia más antigua; los logs del fallo completo se obtuvieron directamente con `kubectl logs catalogo-api`.*

**Conclusión:** El error informado por `pymongo` es **`ServerSelectionTimeoutError`** (`[Errno -2] Name or service not known`) y **NO** `Authentication failed`. Esto ocurre porque `DB_HOST=catalogo-db` ni siquiera pudo resolverse en DNS: en Kubernetes un Pod suelto no provee un nombre resoluble a ningún otro Pod (esa es la función de un **`Service`**, que todavía no existe). Al fallar en la resolución de nombres antes de abrir la conexión TCP, la API nunca alcanzó a MongoDB y por eso jamás llegó a la instancia de autenticación donde aparecería `Authentication failed`.

### 6. Conexión temporal por IP y por qué la IP de un Pod es descartable 

Al reemplazar temporalmente `DB_HOST` en `k8s/pod-catalogo-api.yaml` por la dirección IP original del Pod de MongoDB (`10.244.0.24`) y recrear `catalogo-api`, la API logró conectarse a la base de datos y alcanzó el estado `1/1 Running` sin reinicios.

A continuación se eliminó el Pod `catalogo-db` y se volvió a crear con el mismo manifiesto `k8s/pod-catalogo-db.yaml`. Kubernetes le asignó una dirección IP diferente (`<IP_NUEVA>` en lugar de `10.244.0.24`), provocando que `catalogo-api` volviera a fallar al seguir apuntando a una dirección inexistente.

**Conclusión:** La dirección IP de un Pod es efímera y descartable: cada vez que un Pod se elimina, se reprograma o se recrea, recibe una IP nueva del clúster, por lo que nunca sirve como punto de configuración. En ese hecho reside la razón de existir de los **`Services`** (Etapa 6), que proveen una IP virtual estable y un nombre DNS fijo (`catalogo-db`) por delante de Pods cuyas IPs cambian constantemente.

*Nota de estado final: tras completar las pruebas de la Consigna 8, se restituyó `DB_HOST: "catalogo-db"` en `k8s/pod-catalogo-api.yaml`; el manifiesto queda deliberadamente roto hasta que se cree el `Service` correspondiente en la siguiente entrega.*

### 7. Tercer `host not found in upstream` en `catalogo-frontend`

Al desplegar `k8s/pod-catalogo-frontend.yaml`, Nginx vuelve a fallar al arrancar con el mensaje `host not found in upstream "catalogo-api:8000"`, por tercera vez a lo largo del curso:

| Etapa | Mensaje de error en Nginx | Causa raíz | Cómo se resuelve |
| :---: | :--- | :--- | :--- |
| **Etapa 3** (`docker run`) | `host not found in upstream "catalogo-api:8000"` | **Faltaba la red custom** (`catalogo-net`) con DNS interno embebido, o la API aún no había sido creada. | Crear la red `bridge` custom `catalogo-net` y levantar `catalogo-api` antes del frontend. |
| **Etapa 4** (`docker compose`) | `host not found in upstream "catalogo-api:8000"` | **Condición de carrera al arrancar:** sin `depends_on`, Compose iniciaba Nginx antes de que existiera `catalogo-api`. | Declarar `depends_on` (con `condition: service_healthy`) en `docker-compose.yml`. |
| **Etapa 5** (Kubernetes — Pods sueltos) | `host not found in upstream "catalogo-api:8000"` | **Falta el `Service`:** aunque `catalogo-api` exista y esté corriendo, un Pod suelto no registra ningún nombre DNS en CoreDNS. | Crear un objeto **`Service`** llamado `catalogo-api` (Etapa 6). |

**Conclusión:** Aunque el mensaje de error de Nginx es idéntico al de la tercera etapa, la causa arquitectónica es distinta: entonces faltaba la red de Docker con DNS interno, mientras que en Kubernetes falta el **`Service`** que exponga el nombre `catalogo-api` dentro de CoreDNS.

### 8. Acceso temporal sin `Service` mediante `kubectl port-forward`

Con `catalogo-api` en `Running` (apuntando momentáneamente a la IP actual de `catalogo-db`), se abrió un túnel local con `kubectl port-forward pod/catalogo-api 8000:8000` y se probaron los endpoints:

```bash
$ curl -s localhost:8000/health
{"status":"ok","db":"ok","version":"v1"}

$ curl -s localhost:8000/productos | head -c 250
[{"sku":999666,"categoria":"Apple","creado_en":"2026-11-23T06:09:47.862000","descripcion":"Clásico de 4.7\" con Touch ID y cuerpo de aluminio.","imagen_hash":"7e23251b780a76a1","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 6 (Gold, 32 GB)"...
```

Al acceder a `http://localhost:8000/docs`, Swagger UI devuelve el HTML inicial (`200 OK`) pero **no logra cargar la definición de la API** porque intenta descargar `/api/openapi.json` y recibe **`404 Not Found`**.

**Conclusión:** El `404` en `/docs` (`/api/openapi.json`) **no es un error**, sino la consecuencia directa de haber configurado `ROOT_PATH=/api` desde la Etapa 3 para funcionar detrás del proxy reverso Nginx: al entrar directo al Pod por `port-forward` no existe ningún proxy delante que maneje el prefijo `/api`. Además, **`kubectl port-forward` es únicamente una herramienta de depuración temporal y no un mecanismo de publicación**: escucha solo en `127.0.0.1` local, termina al cerrar la terminal y se interrumpe si el Pod se reinicia porque apunta a un Pod concreto y no a un `Service`.

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

| Aspecto | `docker-compose.yml` (Etapa 4) | Manifiestos `k8s/pod-*.yaml` (Etapa 5) | Qué ocurrió |
| :--- | :--- | :--- | :--- |
| **Imágenes OCI** | `build:` local + `image:` en caché del Docker Engine | `image: jmfloress04/catalogo-*:v1` descargadas desde Docker Hub | **Se conservó** la misma imagen sin cambios; cambió la distribución vía registry. |
| **Variables de entorno** | Bloque `environment:` con interpolación desde `.env` | Lista `env:` (`name`/`value` como strings) dentro de cada manifiesto | **Se conservó** el contrato (`DB_*`, `ROOT_PATH=/api`, `API_*`), pero los secretos volvieron inline (**faltan `ConfigMap` y `Secret`**). |
| **Red y resolución DNS** | Red custom `catalogo-net` con DNS interno automático por nombre | Pods sueltos con IP efímera y **sin** registro DNS en CoreDNS | **Desapareció** la resolución por nombre (**falta `Service`**). |
| **Orden de arranque y salud** | `depends_on` (`condition: service_healthy`) + `healthcheck` | **No existe `depends_on`**; `Running` no equivale a `Ready` | **Desapareció** `depends_on` (**faltan `readinessProbe` / `livenessProbe`** + reintentos en la app). |
| **Exposición de puertos** | `ports: ["3000:8080"]` publicaba directo al host | `containerPort` es solo informativo; acceso solo con `port-forward` | **Desapareció** la publicación directa (**faltan `Service` e `Ingress`**). |
| **Persistencia de datos** | Volumen nombrado `catalogo-db-data` montado en `/data/db` | Sin volumen declarado (al recrear `catalogo-db` se pierden los datos) | **Falta almacenamiento persistente (`PV` / `PVC`)**. |
| **Autorreparación y réplicas** | `restart` en un único host | `kubelet` reinicia contenedores, pero un Pod borrado no se recrea | **Faltan `Deployment`, `ReplicaSet` / `StatefulSet`**. |
| **Seguridad (`cap_drop`)** | `cap_drop: [ALL]` + 4 `cap_add` mínimas en `mongo:7` | Sin restricciones de capabilities declaradas todavía | **Falta configurar `securityContext`**. |

**Conclusión:** Esta entrega conserva intactas las imágenes y el contrato de variables de entorno de la Etapa 4, pero expone de forma deliberada todo lo que falta al migrar únicamente a Pods sueltos: nombres DNS estables (`Service`), autorreparación a nivel de Pod (`Deployment`/`StatefulSet`), sondas de salud (`probes`), configuración desacoplada (`ConfigMap`/`Secret`), persistencia (`PV`/`PVC`) y exposición externa (`Ingress`), recursos que se incorporarán progresivamente en las etapas 6 a 8.

## Etapa 6 — Deployments, Services, Rolling Updates y Réplicas sin Estado

### Orden de aplicación (Services antes que Deployments)

Al aplicar `k8s/deploy-catalogo-frontend.yaml` antes de que existiera el Service `catalogo-api`, las dos réplicas del frontend fallaron al arrancar (`0/1 Error` / `CrashLoopBackOff`, con `3` reinicios en 80s):

```text
NAME                                 READY   STATUS   RESTARTS      AGE
catalogo-frontend-7c89f9b558-8n4mz   0/1     Error    3 (59s ago)   80s
catalogo-frontend-7c89f9b558-dp86h   0/1     Error    3 (62s ago)   80s
```

Error registrado en los logs de Nginx (`kubectl logs -n catalogo-app -l app=catalogo-frontend --tail=10`):

```text
nginx: [emerg] host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
```

**Conclusión:**
- En Kubernetes **no existe `depends_on`** como en Docker Compose.
- Nginx resuelve el nombre del `upstream` (`catalogo-api`) en el momento del arranque; si ese nombre no existe en **CoreDNS**, el proceso termina con error.
- El registro DNS de un `Service` existe desde el instante en que se crea el objeto, **aun sin Pods detrás** (no hace falta esperar a que la API esté sana).
- **Regla operativa:** aplicar siempre primero los `Services` (`svc-catalogo-db`, `svc-catalogo-api`, `svc-catalogo-frontend`) y después los `Deployments` (`deploy-catalogo-db`, `deploy-catalogo-api`, `deploy-catalogo-frontend`).

### Verificación de selectors mediante Endpoints

Salida registrada con `kubectl get endpoints -n catalogo-app`:

```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                ENDPOINTS                           AGE
catalogo-api        10.244.0.33:8000,10.244.0.34:8000   3m52s
catalogo-db         10.244.0.32:27017                   4m7s
catalogo-frontend   10.244.0.35:8080,10.244.0.36:8080   3m45s
```

Verificación cruzada con `kubectl get endpointslices -n catalogo-app` y `kubectl get pods -n catalogo-app -o wide`:
- **`catalogo-db`** (`ClusterIP :27017`): vinculado a **1 Pod** (`10.244.0.32` → `catalogo-db-654f9f8fdc-pf548`).
- **`catalogo-api`** (`ClusterIP :8000`): vinculado a **2 Pods** (`10.244.0.33` y `10.244.0.34`).
- **`catalogo-frontend`** (`NodePort 30302 → 8080`): vinculado a **2 Pods** (`10.244.0.35` y `10.244.0.36`).

**Observaciones:**
- Ningún Service muestra `<none>`, lo que confirma que el `spec.selector` de cada Service coincide exactamente con las `labels` (`app: ...`) de los Pods creados por su Deployment (un error en el selector se crearía sin fallar, pero dejaría `<none>` en `ENDPOINTS`).
- El aviso `Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice` es esperado en el clúster (`v1.37.1`), ya que Kubernetes reemplazó el recurso clásico `Endpoints` por `EndpointSlice`.

### Comprobación de autorreparación

#### Service NodePort de depuración para la API (`catalogo-api-nodeport`)

Se aplicó `k8s/svc-catalogo-api-nodeport.yaml` (`NodePort 30502 → 8000`) seleccionando los mismos Pods (`app: catalogo-api`) que el Service interno `ClusterIP`:

```text
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/catalogo-api            ClusterIP   10.97.194.160    <none>        8000/TCP         25m
service/catalogo-api-nodeport   NodePort    10.107.115.69    <none>        8000:30502/TCP   9s
service/catalogo-db             ClusterIP   10.104.156.166   <none>        27017/TCP        25m
service/catalogo-frontend       NodePort    10.105.47.169    <none>        8080:30302/TCP   25m
```

Verificación con `curl -s http://172.16.21.16:30502/health`:

```json
{"status":"ok","db":"ok","version":"v1"}
```

> **Nota importante:** El objeto `catalogo-api-nodeport` (`k8s/svc-catalogo-api-nodeport.yaml`) **no formará parte del chart de Helm de la Etapa 8**; existe únicamente con fines de depuración. En arquitectura normal, la API solo se expone internamente mediante `ClusterIP`.

#### Pod suelto vs. Deployment → ReplicaSet

Con la aplicación abierta en el navegador, se eliminó manualmente una de las dos réplicas de la API (`catalogo-api-7d4455d6dd-2q6ng`):

```text
# Antes:
catalogo-api-7d4455d6dd-2q6ng   1/1     Running   0          35m
catalogo-api-7d4455d6dd-cd886   1/1     Running   0          35m

# Tras ejecutar: kubectl delete pod catalogo-api-7d4455d6dd-2q6ng -n catalogo-app
catalogo-api-7d4455d6dd-cd886   1/1     Running   0          36m
catalogo-api-7d4455d6dd-qdgbp   1/1     Running   0          13s
```

**Observaciones y contraste con Pod suelto:**
- En segundos, el `ReplicaSet` (`catalogo-api-7d4455d6dd`) detectó que había `1` réplica en lugar de las `2` deseadas y creó automáticamente un Pod nuevo (`catalogo-api-7d4455d6dd-qdgbp`, mismo hash de ReplicaSet pero nuevo sufijo y nueva IP).
- La aplicación **no dejó de responder** en ningún momento porque la segunda réplica (`...-cd886`) siguió atendiendo tráfico y el Service actualizó sus `Endpoints` de forma transparente.
- **Contraste:** con Pods sueltos, eliminar `catalogo-api` lo borraba definitivamente y nadie lo recreaba; ahora el `Deployment` + `ReplicaSet` garantizan la autorreparación continua.

### Actualización progresiva (`RollingUpdate`) y `rollback`

1. **Construcción y publicación de `v2` en Docker Hub:**
   Se construyó y publicó `jmfloress04/catalogo-api:v2` con `--build-arg APP_VERSION=v2` (`digest: sha256:af602965b7e853eaab2479f63a96f1a95010f7794f4c3f15b60110b7966feaa2`), ya que el clúster solo descarga imágenes desde el registry.

2. **Actualización progresiva (`v1 → v2`):**
   Al ejecutar `kubectl set image deployment/catalogo-api catalogo-api=jmfloress04/catalogo-api:v2 -n catalogo-app`, el rollout completó sin cortes (`deployment "catalogo-api" successfully rolled out`).
   - En `kubectl get rs -n catalogo-app -l app=catalogo-api` se comprobó que el `ReplicaSet` anterior quedó conservado en `0` réplicas:
     ```text
     NAME                      DESIRED   CURRENT   READY   AGE
     catalogo-api-7d4455d6dd   0         0         0       59m
     catalogo-api-86bd6d7c     2         2         2       58s
     ```
   - `curl -s http://172.16.21.16:30502/health` (y el indicador de versión del frontend) pasaron de `v1` a **`v2`**:
     ```json
     {"status":"ok","db":"ok","version":"v2"}
     ```

3. **Reversión (`rollback` `v2 → v1`):**
   Al ejecutar `kubectl rollout undo deployment/catalogo-api -n catalogo-app`, Kubernetes escaló nuevamente el `ReplicaSet` de `v1` y bajó a `0` el de `v2`, devolviendo la API y el frontend a **`v1`** (`{"status":"ok","db":"ok","version":"v1"}`).

4. **Historial de revisiones (`kubectl rollout history`):**
   ```text
   # Tras actualizar a v2:        # Tras ejecutar rollout undo:
   REVISION  CHANGE-CAUSE         REVISION  CHANGE-CAUSE
   1         <none>               2         <none>
   2         <none>               3         <none>
   ```
   - La columna `CHANGE-CAUSE` en `<none>` es el comportamiento esperado por defecto.
   - Al hacer `rollback`, el historial **no retrocede sino que avanza**: `v1` se reutiliza y pasa a registrarse como **revisión 3** (y si una imagen `v2` no estuviera publicada en Docker Hub, el Pod nuevo quedaría en `ImagePullBackOff` mientras los Pods de `v1` siguen atendiendo sin interrupción hasta corregir o ejecutar `rollout undo`).

### Escalado a 3 réplicas y por qué la API debe ser sin estado (*stateless*)

1. **Escalado de la API y verificación de respuestas:**
   Se escaló `catalogo-api` a 3 réplicas (`kubectl scale deployment catalogo-api --replicas=3 -n catalogo-app`), todas en `1/1 Running`:
   ```text
   NAME                            READY   STATUS    RESTARTS   AGE
   catalogo-api-7d4455d6dd-5h8pj   1/1     Running   0          37s
   catalogo-api-7d4455d6dd-cl5rq   1/1     Running   0          10m
   catalogo-api-7d4455d6dd-glc9j   1/1     Running   0          10m
   ```
   Se realizaron 10 solicitudes consecutivas contra `http://172.16.21.16:30502/productos` respondiendo todas `200 OK`:
   ```text
   200 200 200 200 200 200 200 200 200 200
   ```

2. **Experimento de archivo en disco local entre réplicas:**
   - Escritura en la réplica `catalogo-api-7d4455d6dd-glc9j`:
     ```text
     catalogo-api-7d4455d6dd-glc9j
     -rw-r--r-- 1 appuser appuser 7 Nov 23 15:07 /tmp/foto.bin
     ```
   - Lectura desde otra réplica (`catalogo-api-7d4455d6dd-cl5rq`):
     ```text
     ls: cannot access '/tmp/foto.bin': No such file or directory
     command terminated with exit code 2
     ```

3. **Conclusión (Variante C — API sin estado):**
   - Cada Pod posee su propio sistema de archivos efímero y aislado. Un archivo escrito en el disco de un Pod **no es visible desde ninguna otra réplica** y **desaparece definitivamente cuando el Pod se recrea**.
   - Si `catalogo-api` guardara las imágenes de productos en su disco local con 3 réplicas detrás del Service, las peticiones que cayeran en cualquiera de las otras dos réplicas fallarían al servir esa imagen (y tras un rolling update o autorreparación se perderían todas).
   - Por esa razón, **`catalogo-app` almacena los bytes de cada imagen dentro del documento de MongoDB**, manteniendo la API completamente **sin estado (*stateless*)** y logrando que todas sus réplicas sean intercambiables.

### Limitación conocida: `catalogo-db` sin persistencia

Se verificó experimentalmente que `catalogo-db` corre actualmente como un `Deployment` de `1` réplica **sin volumen persistente**:

1. **Antes de eliminar el Pod de MongoDB:**
   ```bash
   curl -s http://172.16.21.16:30502/productos | grep -o '"sku"' | wc -l
   ```
    → **`13`** productos (`12` demo + `1` cargado a mano con imagen).

2. **Eliminación y recreación automática de la base de datos:**
   ```bash
   kubectl delete pod -l app=catalogo-db -n catalogo-app
   ```
   el Pod `catalogo-db-654f9f8fdc-pf548` fue reemplazado automáticamente por `catalogo-db-654f9f8fdc-bz8lb` con `/data/db` vacío.
   Al consultar nuevamente `/productos` sin reiniciar la API → **`[]`** (**catálogo vacío**).

3. **Reinicio progresivo de la API (`kubectl rollout restart deployment/catalogo-api -n catalogo-app`):**
   Al volver a consultar `/productos` → **`12`** productos. Reapareció únicamente el catálogo de demostración (porque `SEED_DEMO=true` ejecuta la siembra idempotente al arrancar FastAPI), pero **los productos cargados a mano se perdieron definitivamente**.

**Limitación declarada y arquitectura:**
- **Persistencia pendiente (Etapa 7):** Esta limitación es conocida y explícita en la etapa 6 y se resolverá en la **Etapa 7** adjuntando un volumen persistente (**`PersistentVolumeClaim` / `PVC`**) a `/data/db`.
- **No escalar la base de datos:** `catalogo-db` permanece estrictamente en **`replicas: 1`**. Escalarlo (`--replicas=3`) sería aceptado sin error por Kubernetes, pero levantaría tres instancias independientes de MongoDB sin *replica set* detrás del mismo Service, devolviendo catálogos inconsistentes en cada solicitud.

## Etapa 7 — ConfigMaps, Secrets, Almacenamiento e Ingress

### Secret `catalogo-db-secret` (Base64 no es cifrado)
- `k8s/secret-catalogo-db.yaml` se excluyó de Git mediante `.gitignore`.
- Al inspeccionar el objeto con 
```bash
kubectl get secret catalogo-db-secret -n catalogo-app -o jsonpath='{.data.MONGO_INITDB_ROOT_PASSWORD}' | base64 -d
```
se obtiene el valor en texto plano (`catalogo_pass`), demostrando que Kubernetes solo codifica el dato en **Base64** y no lo cifra (además, al aplicar con `kubectl apply` usando `stringData`, el texto plano queda en la anotación `last-applied-configuration`).
- **Protección real:** para proteger efectivamente estas credenciales hace falta habilitar **cifrado en reposo en `etcd`** (junto con políticas RBAC) o bien utilizar un **gestor de secretos externo** (por ejemplo, HashiCorp Vault o AWS/GCP Secrets Manager).

### Conexión a MongoDB: piezas sueltas vs. `MONGO_URI` completa
- **Decisión:** se eligió pasar la configuración en **piezas separadas** (`DB_HOST`, `DB_PORT`, `DB_NAME` desde el ConfigMap `catalogo-api-config`, y `DB_USER`, `DB_PASSWORD` renombradas con `valueFrom.secretKeyRef` desde `catalogo-db-secret`).
- **Justificación:**
  1. **Sin duplicar credenciales:** MongoDB y la API consumen exactamente las mismas claves del único Secret (`MONGO_INITDB_ROOT_USERNAME` / `MONGO_INITDB_ROOT_PASSWORD`); con `MONGO_URI` habría que repetir la contraseña y meter datos no sensibles (`host`, `port`, `db`) dentro del Secret.
  2. **Manejo automático de `?authSource=admin`:** con piezas separadas la API arma la URI y agrega `?authSource=admin` por sí misma; si se escribiera `MONGO_URI` manualmente en el Secret sin ese parámetro, la autenticación fallaría con `Authentication failed` porque el usuario creado por `MONGO_INITDB_ROOT_*` vive en la base `admin`.
- **Verificación real:** como la API posee valores por omisión que coinciden con Compose, que responda no alcanza para validar la inyección; se confirmó mediante `kubectl exec deploy/catalogo-api -n catalogo-app -- env` (y en el frontend con `env | grep API_`).

### Persistencia de MongoDB: `catalogo-db-pvc`

Se creó el PersistentVolumeClaim `catalogo-db-pvc` (`k8s/pvc-catalogo-db.yaml`) solicitando `1Gi` en modo **`ReadWriteOnce` (`RWO`)** y **sin especificar `storageClassName`** para que tome automáticamente la `StorageClass` por defecto del clúster (`local-path`), manteniendo el manifiesto portable entre entornos (`kind`, `minikube`, `kubeadm`).

#### 1. Estado inmediatamente después de aplicar el PVC (antes del Pod)
```text
$ kubectl get pvc,pv -n catalogo-app
NAME                                    STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/catalogo-db-pvc   Pending                                      local-path     <unset>                 7s

$ kubectl describe pvc catalogo-db-pvc -n catalogo-app | tail -5
Events:
  Type    Reason                Age               From                         Message
  ----    ------                ----              ----                         -------
  Normal  WaitForFirstConsumer  6s (x2 over 14s)  persistentvolume-controller  waiting for first consumer to be created before binding
```
El estado `Pending` inicial **no es un error**: la `StorageClass` `local-path` utiliza modo de enlace `WaitForFirstConsumer`, por lo que difiere la creación física del `PersistentVolume` hasta que exista un Pod consumidor y el scheduler defina en qué nodo se ejecutará.

#### 2. Montaje en `catalogo-db` y transición a `Bound`
Se montó `catalogo-db-pvc` en `/data/db` dentro de `k8s/deploy-catalogo-db.yaml` (declarando el reclamo en `spec.template.spec.volumes` a nivel Pod y el montaje en `volumeMounts` a nivel contenedor, junto con `strategy.type: Recreate` para evitar que dos instancias de `mongod` compitan por el mismo volumen `RWO` al reiniciarse). Al crearse el Pod, el aprovisionador dinámico generó el `PV` y los ligó 1 a 1:

```text
$ kubectl get pvc,pv -n catalogo-app
NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/catalogo-db-pvc   Bound    pvc-306954b6-4d37-4a65-9e9d-dcd9b33799b3   1Gi        RWO            local-path     <unset>                 3m50s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-306954b6-4d37-4a65-9e9d-dcd9b33799b3   1Gi        RWO            Delete           Bound    catalogo-app/catalogo-db-pvc   local-path     <unset>                          6s
```

#### 3. Por qué MongoDB no necesita `subPath` y verificación de permisos en `/data/db`
- **Sin `subPath` (diferencia entre motores):** a diferencia de MySQL (Variante B), que aborta la inicialización si encuentra directorios preexistentes (como `lost+found`) en la raíz del volumen y por ello requiere `subPath`, **MongoDB no exige que `/data/db` esté vacío**, de modo que el PVC se monta directamente sobre `/data/db` sin `subPath`.
- **Propietario y permisos:** lo que sí debe verificarse en MongoDB es que el usuario `mongodb` del contenedor tenga permiso de escritura sobre `/data/db` para evitar fallos `Permission denied` en almacenamientos con permisos restrictivos. Con `local-path` del clúster del curso, el directorio queda con permisos `777` y propietario `mongodb`:

```text
$ kubectl exec deploy/catalogo-db -n catalogo-app -- ls -ld /data/db
drwxrwxrwx 5 mongodb root 4096 Nov 23 16:47 /data/db
```

### Verificación de persistencia y cierre de la limitación

Se cargó un producto nuevo a mano con su imagen desde el navegador (`12` demo + `1` manual = **`13`** productos), se eliminó el Pod de MongoDB y se comprobó que el nuevo Pod recreado por el Deployment montó el mismo PVC `catalogo-db-pvc` conservando los datos intactos:

```text
# Antes de eliminar el Pod de MongoDB (12 demo + 1 cargado a mano):
$ curl -s http://172.16.21.16:30302/api/productos | grep -o '"sku"' | wc -l
13

# Eliminación y recreación automática del Pod de la base de datos:
$ kubectl delete pod -l app=catalogo-db -n catalogo-app
pod "catalogo-db-558886cdd5-5hrjt" deleted from catalogo-app namespace

$ kubectl get pods -n catalogo-app -l app=catalogo-db -w
NAME                           READY   STATUS    RESTARTS   AGE
catalogo-db-558886cdd5-7lf2j   1/1     Running   0          11s

# Después de recrearse el Pod (el producto cargado a mano y su imagen permanecen):
$ curl -s http://172.16.21.16:30302/api/productos | grep -o '"sku"' | wc -l
13
```

- **Advertencia sobre el falso positivo de Variante C:** los 12 productos del catálogo de demostración reaparecen siempre porque la siembra (`SEED_DEMO=true`) es idempotente y se ejecuta en cada arranque de la API. La prueba válida de persistencia es el **producto cargado manualmente con su imagen** (`13` productos antes y después, verificado también en el navegador).
- **Cierre de limitación de Etapa 6:** con `catalogo-db-pvc` montado en `/data/db`, queda resuelta la limitación de datos efímeros que el TP-06 había dejado declarada por escrito.

### Exposición HTTP con Ingress: `catalogo-app-ingress`

Se creó el Ingress `catalogo-app-ingress` (`k8s/ingress-catalogo-app.yaml`) con `ingressClassName: nginx`, host `catalogo.local` y **una sola regla**: `/` (`Prefix`) dirigida a `catalogo-frontend:8080`. Se obtuvo la `EXTERNAL-IP` del `nginx-ingress-controller` (`172.16.21.209`, distinta de la IP del nodo `172.16.21.16`) y se agregó `172.16.21.209 catalogo.local` a `/etc/hosts`.

```text
$ kubectl get ingress -n catalogo-app
NAME                   CLASS   HOSTS            ADDRESS         PORTS   AGE
catalogo-app-ingress   nginx   catalogo.local   172.16.21.209   80      4m43s
```

- **Por qué NO se agrega una regla `/api`:** el Nginx interno del contenedor `catalogo-frontend` ya actúa como proxy inverso hacia `catalogo-api:8000`. Duplicar la regla `/api` en el Ingress crearía **dos fuentes de verdad** para la misma decisión de enrutamiento y rompería la paridad con Docker Compose (donde no existe Ingress).
- **Verificación válida (`/api/productos`) vs. falso positivo SPA (`/productos`):** por la directiva `try_files` de la aplicación React (SPA), cualquier ruta inexistente fuera de `/api` devuelve `200 OK` con `index.html`. Un código `200` por sí solo no alcanza para probar que el Ingress llega a la API:

```text
# Ruta válida (/api/productos) -> 200 OK con application/json (datos reales de FastAPI/MongoDB):
$ curl -s -i http://catalogo.local/api/productos | head -c 400
HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: application/json
Content-Length: 4297

[{"sku":999666,"categoria":"Apple",...

# Falso positivo SPA (/productos) -> 200 OK pero con text/html (index.html servido por try_files):
$ curl -s -i http://catalogo.local/productos | head -c 400
HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: text/html
Content-Length: 397

<!doctype html>
<html lang="es">...
```

### Documentación de la API detrás del Ingress
Al abrir `http://catalogo.local/api/docs` en el navegador, Swagger UI carga correctamente apuntando a `/api/openapi.json`. Esto confirma que `ROOT_PATH="/api"` llega efectivamente desde `catalogo-api-config` hasta FastAPI a través de toda la cadena (`Ingress → Service frontend → Nginx → Service API → FastAPI`); si faltara, la página cargaría en blanco sin errores en los logs.

### Inventario de los 11 objetos actuales del proyecto

| # | Objeto | Tipo | Función |
| :-: | :--- | :--- | :--- |
| 1 | `catalogo-db` | `Deployment` | Mantiene `1` réplica de MongoDB 7 con PVC persistente y credenciales desde Secret. |
| 2 | `catalogo-api` | `Deployment` | Mantiene `2` réplicas *stateless* de FastAPI configuradas vía ConfigMap y Secret renombrado. |
| 3 | `catalogo-frontend` | `Deployment` | Mantiene `2` réplicas de React + Nginx no-root (`:8080`) con proxy interno `/api`. |
| 4 | `catalogo-db` | `Service` (`ClusterIP`) | Provee el DNS interno estable `catalogo-db:27017` dentro del clúster. |
| 5 | `catalogo-api` | `Service` (`ClusterIP`) | Provee el DNS interno estable `catalogo-api:8000` y balancea tráfico hacia la API. |
| 6 | `catalogo-frontend` | `Service` (`NodePort`) | Expone el frontend internamente en `:8080` (destino del Ingress) y en `:30302` del nodo. |
| 7 | `catalogo-api-config` | `ConfigMap` | Guarda la configuración no sensible de la API (`DB_HOST`, `DB_PORT`, `DB_NAME`, `ROOT_PATH`, `SEED_DEMO`). |
| 8 | `catalogo-frontend-config` | `ConfigMap` | Guarda la configuración no sensible del frontend (`API_HOST`, `API_PORT`) para `envsubst`. |
| 9 | `catalogo-db-secret` | `Secret` (`Opaque`) | Guarda `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD` compartidos por BD y API (fuera de Git). |
| 10 | `catalogo-db-pvc` | `PersistentVolumeClaim` | Reclama `1Gi` (`ReadWriteOnce`) montado en `/data/db` para persistir MongoDB. |
| 11 | `catalogo-app-ingress` | `Ingress` | Expone `http://catalogo.local/` (puerto 80) con una única regla hacia `catalogo-frontend:8080`. |

*(En la Etapa 8 se incorporará el 12.° objeto: el autoescalador `HPA`. Los Pods sueltos y el Service auxiliar `catalogo-api-nodeport` de depuración no forman parte del inventario productivo.)*

## Etapa 8 — Escalado Automático (HPA) y Empaquetado con Helm

### 1. Declaración de `resources.requests` y `limits` en `catalogo-api`
Antes de crear el `HorizontalPodAutoscaler`, se verificó el funcionamiento de `metrics-server` (`kubectl top nodes` y `kubectl top pods -n catalogo-app`) y se midió el consumo base en reposo de `catalogo-api` (`2m` de CPU y `40–46 Mi` de RAM por réplica, sobre un nodo con ~2 vCPU y ~1.9 GiB de RAM).

En base a esa medición se declararon en `k8s/deploy-catalogo-api.yaml`:
- **`requests` (`cpu: 100m`, `memory: 64Mi`):** define el mínimo garantizado para el *scheduler* y constituye el **denominador exacto** contra el cual el HPA calcula el porcentaje de CPU (un objetivo del `50 %` equivale a `50m` por Pod, no al 50 % de `limits` ni del nodo). Sin `resources.requests`, el HPA permanece indefinidamente en `<unknown>`.
- **`limits` (`cpu: 250m`, `memory: 128Mi`):** protege al nodo evitando que un pico o fallo de un Pod consuma todos los recursos (CPU con *throttling* y memoria acotada frente a un nodo que ya opera al 74 % de RAM, permitiendo escalar hasta 6 réplicas sin dejar Pods en `Pending`).

Verificación aplicada en el clúster:
```text
$ kubectl describe deploy catalogo-api -n catalogo-app | grep -A6 "Limits:"
    Limits:
      cpu:     250m
      memory:  128Mi
    Requests:
      cpu:     100m
      memory:  64Mi
```

### 2. Objeto N° 12: Creación del `HorizontalPodAutoscaler` (`catalogo-api-hpa`)
Se creó `k8s/hpa-catalogo-api.yaml` (`autoscaling/v2`) apuntando al Deployment `catalogo-api` con rango `[2, 6]` réplicas y objetivo `averageUtilization: 50` sobre CPU.

Verificación de métrica activa (sin `<unknown>`):
```text
$ kubectl get hpa -n catalogo-app
NAME               REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%   2         6         2          53s
```

En `kubectl describe hpa catalogo-api-hpa -n catalogo-app` se confirma explícitamente contra qué valor calcula el porcentaje:
- `resource cpu on pods (as a percentage of request): 2% (2m) / 50%` → los `2m` en reposo sobre `requests.cpu: 100m` dan exactamente `2 %` (`ScalingActive: True`, `Reason: ValidMetricFound`).
- **¿Por qué hay un único HPA entre 12 objetos?**
  - **`catalogo-api` (sí):** capa *stateless* cuya CPU escala con las solicitudes HTTP y la recuperación de imágenes desde MongoDB; el HPA no crea Pods directamente, sino que ajusta `spec.replicas` del Deployment dentro de `[2, 6]` (mínimo `2` para alta disponibilidad y *rolling updates*; máximo `6` para proteger al nodo).
  - **`catalogo-frontend` (no):** aunque es *stateless*, Nginx sirviendo estáticos consume `0m` de CPU en reposo (`2–4 Mi` de RAM) y casi no se mueve bajo carga; un HPA indemostrable no aporta valor.
  - **`catalogo-db` (no):** capa *stateful* con PVC `ReadWriteOnce` (`RWO`); múltiples réplicas de un Deployment competirían por el mismo volumen `/data/db`. Escalar la BD requiere un `StatefulSet` / Replica Set.

### 3. Demostración bajo carga real (`2 → 4 → 6 → 3 → 2`)
Se generó carga real dentro del clúster en dos etapas: primero con un Pod temporal `busybox` sobre `http://catalogo-api:8000/productos` y luego con **3 Pods simultáneos (`carga-1`, `carga-2`, `carga-3`)** consultando el endpoint de imágenes `http://catalogo-api:8000/productos/6b046e42a1573dba71b9c69d/imagen` (que obliga a FastAPI a recuperar los bytes de la imagen desde MongoDB y servirlos). Finalmente se eliminaron los Pods de carga y se esperó la ventana de estabilización.

Secuencia capturada con `kubectl get hpa catalogo-api-hpa -n catalogo-app -w`:
```text
NAME               REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%   2         6         2          5m9s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 8%/50%   2         6         2          6m30s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 208%/50%   2         6         2          6m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 188%/50%   2         6         4          7m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 102%/50%   2         6         6          7m15s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 38%/50%    2         6         6          7m30s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         6          7m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 42%/50%    2         6         6          8m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 122%/50%   2         6         6          9m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 123%/50%   2         6         6          9m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 120%/50%   2         6         6          10m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 114%/50%   2         6         6          10m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 19%/50%    2         6         6          10m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         6          10m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         6          15m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         3          15m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         2          15m
```

Eventos registrados en `kubectl describe hpa catalogo-api-hpa -n catalogo-app`:
```text
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  10m   horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  10m   horizontal-pod-autoscaler  New size: 6; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  2m1s  horizontal-pod-autoscaler  New size: 3; reason: All metrics below target
  Normal  SuccessfulRescale  106s  horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
```

**Observaciones técnicas comprobadas en vivo:**
1. **Escalado rápido hacia arriba y techo `maxReplicas`:** al alcanzar `208 %` sobre 2 réplicas ($\lceil 2 \times 208 / 50 \rceil = 9$), el HPA escaló en 30 segundos (`2 → 4 → 6`) y respetó el tope `maxReplicas: 6` incluso cuando los 3 Pods sobre `/imagen` sostuvieron `120–123 %` de CPU, protegiendo al nodo (las 6 réplicas consumieron `39–41 Mi` c/u sin `OOMKilled` ni Pods `Pending`).
2. **Ventana de estabilización de *scale-down* (300 s) y piso `minReplicas`:** al retirar la carga en el minuto `10m` (`cpu: 2%/50%`), el HPA mantuvo las 6 réplicas durante exactamente **5 minutos** hasta el minuto `15m` para evitar *flapping*, y luego redujo gradualmente (`6 → 3 → 2`), deteniéndose en `minReplicas: 2` (`ScalingLimited: True / TooFewReplicas`) para preservar disponibilidad.
3. **¿Qué deja de estar garantizado al pasar de 1 réplica de la API a 3 (o 6)?** Deja de estar garantizado cualquier dato guardado en la memoria local o en el sistema de archivos del Pod de `catalogo-api`: el Service balancea cada pedido a una réplica distinta y las réplicas se crean y destruyen dinámicamente. Por eso en `catalogo-app` los productos **y los bytes de sus imágenes** residen en MongoDB (`catalogo-db` + `catalogo-db-pvc`), y la siembra `SEED_DEMO=true` es idempotente (`upsert` con índice único sobre `sku`) para evitar duplicados cuando arrancan múltiples réplicas en paralelo.

### 4 y 5. Chart de Helm `catalogo-app/` (12 objetos exactos) y parametrización efectiva
Se creó el Chart de Helm en `catalogo-app/` conservando intacta la carpeta `k8s/` del recorrido previo. El chart contiene exactamente los **12 objetos productivos** (excluyendo el Service de depuración `catalogo-api-nodeport` de TP-06 y los Pods sueltos de TP-05):
- 3 `Deployments` (`catalogo-frontend`, `catalogo-api`, `catalogo-db`)
- 3 `Services` (`catalogo-frontend` NodePort `30302`, `catalogo-api` y `catalogo-db` ClusterIP)
- 2 `ConfigMaps` (`catalogo-frontend-config`, `catalogo-api-config`)
- 1 `Secret` (`catalogo-db-secret`)
- 1 `PersistentVolumeClaim` (`catalogo-db-pvc`)
- 1 `Ingress` (`catalogo-app-ingress`, host `catalogo.local → catalogo-frontend:8080`)
- 1 `HorizontalPodAutoscaler` (`catalogo-api-hpa`)

**Parametrización en `values.yaml` (y prueba anti-parametrización decorativa):**
Se parametrizaron repositorios y tags de las imágenes propias (`api.image.tag`, `frontend.image.tag`), réplicas, puertos, `resources.requests`/`limits`, umbrales del HPA (`minReplicas`, `maxReplicas`, `targetCPUUtilizationPercentage`), configuración de ambos ConfigMaps, almacenamiento del PVC, host del Ingress y credenciales (con placeholders en `values.yaml` y valores reales en `catalogo-app/values-secret.yaml`, ignorado en `.gitignore`).

Validación de `helm lint`, conteo exacto en orden fijo y verificación de cambio efectivo mediante `--set`:
```text
$ helm lint ./catalogo-app -f catalogo-app/values-secret.yaml
1 chart(s) linted, 0 chart(s) failed

$ helm template catalogo-app ./catalogo-app -n catalogo-app -f catalogo-app/values-secret.yaml | grep '^kind:' | uniq -c
      1 kind: Secret
      2 kind: ConfigMap
      1 kind: PersistentVolumeClaim
      3 kind: Service
      3 kind: Deployment
      1 kind: HorizontalPodAutoscaler
      1 kind: Ingress

$ helm template catalogo-app ./catalogo-app -n catalogo-app \
    --set api.image.tag=v99 --set frontend.image.tag=v88 \
    --set api.replicaCount=5 --set hpa.maxReplicas=9 \
    | grep -E '(image:|replicas:|maxReplicas:)'
  replicas: 5
          image: "jmfloress04/catalogo-api:v99"
  replicas: 1
          image: "mongo:7"
  replicas: 2
          image: "jmfloress04/catalogo-frontend:v88"
  maxReplicas: 9
```

- **Orden fijo por tipo de objeto:** Helm aplica `Secret → ConfigMap → PVC → Service → Deployment → HPA → Ingress`. Al crear los `Services` antes que los `Deployments`, desaparece automáticamente el error `host not found in upstream` observado en la Etapa 6 cuando Nginx arrancaba antes de existir el DNS `catalogo-api`.
- **Decisión de conexión a MongoDB (Variante C):** se sostiene el modelo de **piezas separadas** (`DB_HOST`, `DB_PORT`, `DB_NAME` en `catalogo-api-config` + `MONGO_INITDB_ROOT_USERNAME` / `MONGO_INITDB_ROOT_PASSWORD` en `catalogo-db-secret`, renombradas a `DB_USER` / `DB_PASSWORD` vía `valueFrom.secretKeyRef`). Así existe una única fuente de verdad para la contraseña, no se duplica ni se mezcla con configuración pública, y `backend/main.py` agrega `?authSource=admin` automáticamente (evitando el fallo `Authentication failed` que ocurre al componer una `MONGO_URI` incompleta o partida entre ConfigMap y Secret).

### 6. Instalación limpia de la Release con `helm install --create-namespace`
Para evitar colisión con los manifiestos aplicados manualmente en TP-05..TP-07, se eliminó el namespace previo (`kubectl delete namespace catalogo-app`) y se instalaron los 12 objetos en un único comando:

```bash
helm install catalogo-app ./catalogo-app \
  -n catalogo-app --create-namespace \
  -f catalogo-app/values-secret.yaml
```

Verificación completa de los 12 objetos del proyecto (`kubectl get all,cm,secret,pvc,ingress,hpa -n catalogo-app`):
```text
NAME                                     READY   STATUS    RESTARTS   AGE
pod/catalogo-api-5d9495fb65-7snn2        1/1     Running   0          47s
pod/catalogo-api-5d9495fb65-w8j85        1/1     Running   0          47s
pod/catalogo-db-558886cdd5-5grxx         1/1     Running   0          47s
pod/catalogo-frontend-5c4996b696-5bwd5   1/1     Running   0          47s
pod/catalogo-frontend-5c4996b696-hr8xm   1/1     Running   0          47s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/catalogo-api        ClusterIP   10.99.95.19      <none>        8000/TCP         47s
service/catalogo-db         ClusterIP   10.105.193.192   <none>        27017/TCP        47s
service/catalogo-frontend   NodePort    10.97.129.4      <none>        8080:30302/TCP   47s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalogo-api        2/2     2            2           47s
deployment.apps/catalogo-db         1/1     1            1           47s
deployment.apps/catalogo-frontend   2/2     2            2           47s

NAME                                                   REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/catalogo-api-hpa   Deployment/catalogo-api   cpu: 3%/50%   2         6         2          47s

NAME                                 DATA   AGE
configmap/catalogo-api-config        5      47s
configmap/catalogo-frontend-config   2      47s
configmap/kube-root-ca.crt           1      47s

NAME                                        TYPE                 DATA   AGE
secret/catalogo-db-secret                   Opaque               2      48s
secret/sh.helm.release.v1.catalogo-app.v1   helm.sh/release.v1   1      48s

NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/catalogo-db-pvc   Bound    pvc-1f72bfb7-2242-40aa-829f-a93f1cfa398d   1Gi        RWO            local-path     48s

NAME                                             CLASS   HOSTS            ADDRESS         PORTS   AGE
ingress.networking.k8s.io/catalogo-app-ingress   nginx   catalogo.local   172.16.21.209   80      48s
```
*(Además de los 12 objetos del proyecto, aparecen los recursos automáticos del clúster/Helm: `configmap/kube-root-ca.crt` y `secret/sh.helm.release.v1.catalogo-app.v1`, donde Helm almacena el estado de la revisión 1.)*

Verificación funcional detrás del Ingress:
- `helm list -n catalogo-app` → `REVISION: 1`, `STATUS: deployed`, `CHART: catalogo-app-0.1.0`, `APP VERSION: v1`.
- `http://catalogo.local/api/productos` → `HTTP/1.1 200 OK`, `Content-Type: application/json`.
- `http://catalogo.local/api/docs` → `url: '/api/openapi.json'` (confirma `ROOT_PATH="/api"` desde `catalogo-api-config`).
- `kubectl get hpa -n catalogo-app` → `cpu: 2%/50%` (sin `<unknown>`, `0` reinicios en todos los Pods).

### 7. Ciclo completo de Release: `helm upgrade` (`v2`), `helm history` y `helm rollback` (`v1`)
Se ejecutó el ciclo completo de actualización y reversión sobre la release `catalogo-app` pasando `-f catalogo-app/values-secret.yaml` (ya que `helm upgrade` no hereda overrides previos salvo que se repitan o se use `--reuse-values`):

1. **`helm upgrade` modificando solo el tag de la API a `v2` (Revisión `2`):**
   ```bash
   helm upgrade catalogo-app ./catalogo-app \
     -n catalogo-app \
     -f catalogo-app/values-secret.yaml \
     --set api.image.tag=v2
   ```
   Imágenes desplegadas tras el `upgrade`:
   ```text
   NAME                IMAGE
   catalogo-api        jmfloress04/catalogo-api:v2
   catalogo-db         mongo:7
   catalogo-frontend   jmfloress04/catalogo-frontend:v1
   ```
   - **Observación obligatoria (ciclos de vida independientes):** modificar `api.image.tag` de `v1` a `v2` **no modifica el tag de `catalogo-frontend`** (que permanece en `v1`), porque backend y frontend son dos imágenes independientes con ciclos de despliegue independientes.

2. **`helm rollback catalogo-app 1 -n catalogo-app` y consulta de historial (`helm history`):**
   ```text
   REVISION  UPDATED                   STATUS      CHART               APP VERSION  DESCRIPTION
   1         Mon Nov 23 19:35:19 2026  superseded  catalogo-app-0.1.0  v1           Install complete
   2         Mon Nov 23 19:41:02 2026  superseded  catalogo-app-0.1.0  v1           Upgrade complete
   3         Mon Nov 23 19:42:40 2026  deployed    catalogo-app-0.1.0  v1           Rollback to 1
   ```
   - El rollback **no borra la revisión `2`**: crea la **revisión `3`** con el estado completo de la `1` (análogo a `git revert`), devolviendo `catalogo-api` a `jmfloress04/catalogo-api:v1` y revirtiendo los 12 objetos como una única unidad (confirmado en el indicador de versión del navegador: `v1 → v2 → v1`).