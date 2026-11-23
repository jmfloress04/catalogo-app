# Arquitectura — `catalogo-app` (Variante C)

Documento de decisiones de arquitectura para **`catalogo-app`** (**React + Vite** servido por **Nginx unprivileged** · **Python + FastAPI** · **MongoDB 7**), correspondiente al Proyecto Final de *Contenedores, Docker y Orquestación con Kubernetes* (FACET – UNT).

Cada respuesta se conecta directamente con la **etapa del curso** en [README.md](README.md), con los **archivos reales** del repositorio y con la **evidencia ejecutable** en [INFORME.md](INFORME.md), de modo que sirva tanto para la evaluación como de apunte trazable de todo el recorrido.

### Mapa de preguntas y etapas relacionadas

| Sección | Pregunta / Tema | Etapas de referencia en `README.md` |
| :--- | :--- | :--- |
| [1a](#1a-por-qué-el-frontend-se-configura-con-api_host-en-tiempo-de-ejecución-y-no-con-vite_api_url-en-tiempo-de-construcción-si-fuera-al-revés-qué-habría-que-rehacer-para-pasar-de-desarrollo-a-producción) | `API_HOST` en runtime vs. `VITE_API_URL` en build-time | [Etapa 2](README.md#etapa-2--medición-y-optimización-de-imágenes), [Etapa 3](README.md#frontend-conectado-a-catalogo-net-y-verificación-del-reverse-proxy), [Etapa 7](README.md#etapa-7--configmaps-secrets-almacenamiento-e-ingress), [Etapa 8](README.md#4-y-5-chart-de-helm-catalogo-app-12-objetos-exactos-y-parametrización-efectiva) |
| [1b](#1b-por-qué-esta-arquitectura-no-necesita-cors-qué-habría-que-agregar-si-la-aplicación-de-página-única-se-comunicara-directamente-con-la-api) | Ausencia de CORS y arquitectura *reverse proxy* | [Etapa 1](README.md#etapa-1--arquitectura), [Etapa 3](README.md#verificación-del-proxy-y-efecto-de-root_path-en-fastapi), [Etapa 7](README.md#exposición-http-con-ingress-catalogo-app-ingress) |
| [1c](#1c-por-qué-hay-un-solo-autoescalador-entre-los-doce-objetos-qué-ocurriría-si-se-le-asignara-uno-a-catalogo-db-y-por-qué-tampoco-lo-tiene-el-frontend) | Único HPA (`catalogo-api`), por qué no en DB ni frontend | [Etapa 6](README.md#limitación-conocida-catalogo-db-sin-persistencia), [Etapa 7](README.md#persistencia-de-mongodb-catalogo-db-pvc), [Etapa 8](README.md#2-objeto-n-12-creación-del-horizontalpodautoscaler-catalogo-api-hpa) |
| [2](#2-pregunta-propia-de-la-variante-c-qué-deja-de-estar-garantizado-al-pasar-de-una-réplica-de-la-api-a-tres) | Variante C: API *stateless* al escalar `1 → 3` réplicas | [Etapa 5](README.md#9-mortalidad-de-los-pods-sueltos), [Etapa 6](README.md#escalado-a-3-réplicas-y-por-qué-la-api-debe-ser-sin-estado-stateless), [Etapa 7](README.md#verificación-de-persistencia-y-cierre-de-la-limitación), [Etapa 8](README.md#3-demostración-bajo-carga-real-2--4--6--3--2) |
| [3](#3-dificultades-propias-de-mongodb-presentadas-durante-el-curso) | Dificultades reales de MongoDB durante el curso | [Etapa 3](README.md#diagnóstico-de-fallas-resolución-de-nombre-vs-autenticación), [Etapa 4](README.md#healthcheck-de-catalogo-db-mongodb-7), [Etapa 7](README.md#conexión-a-mongodb-piezas-sueltas-vs-mongo_uri-completa) |
| [4](#4-qué-quedaría-pendiente-para-un-entorno-de-producción-real) | Pendientes para producción real y `Deployment` vs. Replica Set | [Etapa 2](README.md#efecto-del-cache-de-construcción), [Etapa 4](README.md#revisión-de-seguridad-y-escaneo-de-vulnerabilidades-trivy), [Etapa 7](README.md#secret-catalogo-db-secret-base64-no-es-cifrado), [Etapa 8](README.md#7-ciclo-completo-de-release-helm-upgrade-v2-helm-history-y-helm-rollback-v1) |

---

## 1. Preguntas obligatorias del curso

### 1a. ¿Por qué el frontend se configura con `API_HOST` en tiempo de ejecución y no con `VITE_API_URL` en tiempo de construcción? Si fuera al revés, ¿qué habría que rehacer para pasar de desarrollo a producción?

> **Dónde se construyó y probó en el curso:** [Etapa 2 — Medición y Optimización de Imágenes](README.md#etapa-2--medición-y-optimización-de-imágenes) (imagen multietapa de `73,9 MB`), [Etapa 3 — Frontend conectado a `catalogo-net`](README.md#frontend-conectado-a-catalogo-net-y-verificación-del-reverse-proxy), [Etapa 4 — Docker Compose](README.md#etapa-4--docker-compose-y-seguridad-básica), [Etapa 7 — ConfigMaps](README.md#etapa-7--configmaps-secrets-almacenamiento-e-ingress) y [Etapa 8 — Chart de Helm](README.md#4-y-5-chart-de-helm-catalogo-app-12-objetos-exactos-y-parametrización-efectiva).

En [`frontend/Dockerfile`](frontend/Dockerfile), la etapa `builder` ejecuta `npm run build` y copia únicamente el resultado estático (`/build/dist`) a la imagen final `nginxinc/nginx-unprivileged:1.27-alpine`, junto con la plantilla [`frontend/nginx/templates/default.conf.template`](frontend/nginx/templates/default.conf.template):

```nginx
location /api/ {
    proxy_pass http://${API_HOST}:${API_PORT}/;
}
```

El código React compilado **no sabe la dirección del backend**: siempre llama a rutas relativas (`/api/...`). Al arrancar el contenedor, el script `/docker-entrypoint.d/20-envsubst-on-templates.sh` de Nginx sustituye únicamente las variables que coinciden con `NGINX_ENVSUBST_FILTER=^API_` (`API_HOST` y `API_PORT`) generando `/etc/nginx/conf.d/default.conf` en **tiempo de ejecución**.

**Causa arquitectónica (*build once, deploy anywhere*):** separar el artefacto inmutable (la imagen) de su configuración permite promover exactamente la **misma imagen** `jmfloress04/catalogo-frontend:v1` —con el mismo digest— a través de todas las etapas que recorrimos en el curso sin reconstruirla jamás:
- En [Etapa 3](README.md#frontend-conectado-a-catalogo-net-y-verificación-del-reverse-proxy) y [Etapa 4](README.md#etapa-4--docker-compose-y-seguridad-básica) (`docker-compose.yml`), `API_HOST` se inyectó por variable de entorno apuntando al DNS interno de `catalogo-net`.
- En [Etapa 5](README.md#7-tercer-host-not-found-in-upstream-en-catalogo-frontend) pudo apuntar incluso a una IP temporal de Pod, y en [Etapa 7](README.md#etapa-7--configmaps-secrets-almacenamiento-e-ingress) pasó a leerse desde el ConfigMap [`k8s/configmap-catalogo-frontend.yaml`](k8s/configmap-catalogo-frontend.yaml).
- En [Etapa 8](README.md#4-y-5-chart-de-helm-catalogo-app-12-objetos-exactos-y-parametrización-efectiva), el mismo valor se parametriza por entorno en [`catalogo-app/values.yaml`](catalogo-app/values.yaml) (`frontend.config.apiHost`).

**Qué habría que rehacer si se hubiera usado `VITE_API_URL` en tiempo de construcción:**
Vite sustituye las variables `VITE_*` literalmente dentro de los archivos `.js` minificados durante `npm run build` (en el `builder`). Si la dirección de la API quedara horneada en build-time, para pasar de desarrollo a producción no alcanzaría con cambiar un `ConfigMap` o un valor en `values.yaml`; habría que:
1. Recompilar la SPA con Vite (`npm run build`) inyectando la nueva `VITE_API_URL` como `ARG`.
2. Reconstruir la imagen de contenedor (`docker build`), generando un binario distinto para cada entorno.
3. Volver a etiquetar y publicar esa imagen nueva en el registry (`docker push`).
4. Repetir escaneos y pruebas (como los de [Etapa 4 — Trivy](README.md#revisión-de-seguridad-y-escaneo-de-vulnerabilidades-trivy)), porque el artefacto que llegaría a producción **no sería el mismo** que se validó en desarrollo/staging.

---

### 1b. ¿Por qué esta arquitectura no necesita CORS? ¿Qué habría que agregar si la aplicación de página única se comunicara directamente con la API?

> **Dónde se construyó y probó en el curso:** [Etapa 1 — Arquitectura](README.md#etapa-1--arquitectura), [Etapa 3 — Verificación del proxy y efecto de `ROOT_PATH`](README.md#verificación-del-proxy-y-efecto-de-root_path-en-fastapi) y [Etapa 7 — Exposición HTTP con Ingress](README.md#exposición-http-con-ingress-catalogo-app-ingress).

**Por qué no necesita CORS:**
CORS (*Cross-Origin Resource Sharing*) es una restricción que impone **el navegador web** (por la *Same-Origin Policy*) cuando código JavaScript intenta leer respuestas HTTP de un origen `(esquema, host, puerto)` diferente de aquel desde donde se descargó la página.

Desde la [Etapa 1 — Arquitectura](README.md#etapa-1--arquitectura) definimos a `catalogo-frontend` como el **único punto de entrada** para el usuario:
- En Docker Compose ([Etapa 4](README.md#etapa-4--docker-compose-y-seguridad-básica)), el navegador solo habla con `http://localhost:3000`.
- En Kubernetes ([Etapa 7 — Exposición HTTP con Ingress](README.md#exposición-http-con-ingress-catalogo-app-ingress)), [`k8s/ingress-catalogo-app.yaml`](k8s/ingress-catalogo-app.yaml) define una **única regla `/`** hacia `catalogo-frontend:8080`, de modo que el navegador solo habla con `http://catalogo.local` (tanto para el HTML/JS como para `/api/productos` y `/api/productos/{id}/imagen`).

Al compartir exactamente el mismo origen, el navegador **nunca dispara comprobaciones CORS**. El salto desde `/api/...` hacia `http://catalogo-api:8000/...` ocurre dentro de Nginx ([`default.conf.template`](frontend/nginx/templates/default.conf.template)) en tráfico servidor-a-servidor sobre la red interna, donde CORS no existe. Como beneficio adicional, `catalogo-api` permanece aislada como `Service` [`ClusterIP`](k8s/svc-catalogo-api.yaml) sin exponerse al exterior.

**Qué habría que agregar si la SPA se comunicara directamente con la API:**
Si el navegador llamara a otro origen (por ejemplo desde `http://catalogo.local` hacia `http://api.catalogo.local` o `http://localhost:8000`):
1. **Exponer la API hacia el cliente externo:** publicar `catalogo-api` mediante una regla/host propio en el `Ingress` o un `Service` alcanzable desde el navegador (aumentando la superficie de ataque).
2. **Registrar `CORSMiddleware` en [`backend/main.py`](backend/main.py):** configurar `fastapi.middleware.cors.CORSMiddleware` indicando explícitamente:
   - `allow_origins`: los orígenes exactos del frontend (evitando `*` en producción o si se envían credenciales).
   - `allow_methods`: los verbos REST usados (`GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`).
   - `allow_headers`: encabezados como `Content-Type` o `Authorization`, imprescindibles para que el navegador apruebe las peticiones *preflight* (`OPTIONS`) previas a `POST`/`PUT`/`DELETE` con JSON.
   - `allow_credentials`, si existiera autenticación por cookies/sesión.
3. **Configurar la URL pública de la API en el navegador:** el código cliente necesitaría conocer un dominio público resoluble fuera del clúster (ya no serviría el DNS interno `catalogo-api`).

---

### 1c. ¿Por qué hay un solo autoescalador entre los doce objetos? ¿Qué ocurriría si se le asignara uno a `catalogo-db`? ¿Y por qué tampoco lo tiene el frontend?

> **Dónde se construyó y probó en el curso:** [Etapa 6 — Limitación conocida de `catalogo-db`](README.md#limitación-conocida-catalogo-db-sin-persistencia), [Etapa 7 — Persistencia de MongoDB (`catalogo-db-pvc`)](README.md#persistencia-de-mongodb-catalogo-db-pvc), [Etapa 8 — `resources.requests` y `limits`](README.md#1-declaración-de-resourcesrequests-y-limits-en-catalogo-api), [Etapa 8 — Objeto N° 12 (`catalogo-api-hpa`)](README.md#2-objeto-n-12-creación-del-horizontalpodautoscaler-catalogo-api-hpa) y [INFORME.md — Sección 2](INFORME.md#2-escalado-automático-bajo-carga-kubectl-get-hpa--w).

Entre los 12 objetos de `k8s/` (y del chart `catalogo-app/`), el único `HorizontalPodAutoscaler` ([`k8s/hpa-catalogo-api.yaml`](k8s/hpa-catalogo-api.yaml)) escala **`Deployment/catalogo-api`** porque es la única capa que reúne simultáneamente dos condiciones:
1. Es **sin estado (*stateless*)** (como se comprobó en [Etapa 6](README.md#escalado-a-3-réplicas-y-por-qué-la-api-debe-ser-sin-estado-stateless)).
2. Su consumo de **CPU crece directamente con las solicitudes** (validación Pydantic, serialización JSON y lectura/entrega de imágenes desde MongoDB). Como medimos en [Etapa 8 — Demostración bajo carga real](README.md#3-demostración-bajo-carga-real-2--4--6--3--2), con `requests.cpu: 100m` en [`k8s/deploy-catalogo-api.yaml`](k8s/deploy-catalogo-api.yaml), `metrics-server` reportó `2 %` en reposo y `208 %` bajo carga, disparando el escalado automático `2 → 4 → 6` y el repliegue `6 → 3 → 2`.

**Qué ocurriría si se le asignara un HPA a `catalogo-db`:**
En [Etapa 7 — Persistencia de MongoDB](README.md#persistencia-de-mongodb-catalogo-db-pvc) montamos [`k8s/pvc-catalogo-db.yaml`](k8s/pvc-catalogo-db.yaml) con modo **`ReadWriteOnce` (`RWO`)** y declaramos explícitamente `replicas: 1` y `strategy: Recreate` en [`k8s/deploy-catalogo-db.yaml`](k8s/deploy-catalogo-db.yaml). Si un HPA intentara escalar ese Deployment a `2` o más réplicas:
- **En nodos distintos:** un volumen `RWO` solo puede montarse en lectura/escritura en un nodo a la vez; las réplicas nuevas quedarían trabadas indefinidamente en `ContainerCreating` (`Multi-Attach error`).
- **En el mismo nodo:** dos procesos `mongod` abriendo el mismo `/data/db` chocan por el archivo de bloqueo `mongod.lock` (provocando `CrashLoopBackOff`) o generan **corrupción de los archivos de WiredTiger**.
- **Si cada Pod tuviera disco propio sin replicación:** tal como se advirtió en [Etapa 6 — Limitación conocida](README.md#limitación-conocida-catalogo-db-sin-persistencia), Kubernetes repartiría consultas entre varias instancias independientes de MongoDB sin *Replica Set*, devolviendo catálogos inconsistentes según qué réplica atendiera cada request (*split-brain*).

**Por qué tampoco lo tiene `catalogo-frontend`:**
Aunque `catalogo-frontend` es *stateless*, Nginx sirviendo archivos estáticos (`/usr/share/nginx/html`) y actuando como proxy ligero consume prácticamente `0m` de CPU (`2–4 Mi` de RAM) tanto en reposo como bajo carga moderada (ver [Etapa 8 — Objeto N° 12](README.md#2-objeto-n-12-creación-del-horizontalpodautoscaler-catalogo-api-hpa)). El cuello de botella del sistema está en la API y la base de datos, no en Nginx: un HPA sobre el frontend sería indemostrable por CPU y añadiría complejidad sin beneficio real. Con `replicas: 2` fijas en [`k8s/deploy-catalogo-frontend.yaml`](k8s/deploy-catalogo-frontend.yaml) ya queda cubierta la disponibilidad y las actualizaciones sin corte.

---

## 2. Pregunta propia de la Variante C: ¿qué deja de estar garantizado al pasar de una réplica de la API a tres?

> **Dónde se construyó y probó en el curso:** [Etapa 5 — Mortalidad de los Pods sueltos](README.md#9-mortalidad-de-los-pods-sueltos), [Etapa 6 — Comprobación de autorreparación](README.md#comprobación-de-autorreparación), [Etapa 6 — Escalado a 3 réplicas y por qué la API debe ser sin estado](README.md#escalado-a-3-réplicas-y-por-qué-la-api-debe-ser-sin-estado-stateless), [Etapa 7 — Verificación de persistencia](README.md#verificación-de-persistencia-y-cierre-de-la-limitación) y **[INFORME.md — Sección 5](INFORME.md#5-prueba-de-la-sexta-entrega-3-réplicas-de-la-api-y-sistema-de-archivos-aislado-stateless)**.

Al pasar `catalogo-api` de `1` a `3` réplicas (y hasta `6` bajo el HPA de [Etapa 8](README.md#3-demostración-bajo-carga-real-2--4--6--3--2)), **deja de estar garantizado que dos peticiones sucesivas lleguen al mismo contenedor y, por ende, cualquier dato almacenado en la memoria RAM o en el sistema de archivos local del Pod**.

La causa llega hasta el modelo de ejecución de Kubernetes:
1. **Aislamiento de filesystem entre réplicas:** cada Pod de `catalogo-api` posee su propia capa escribible efímera (*overlayfs*). El `Service` `catalogo-api` balancea cada solicitud a cualquiera de los Endpoints activos: **un archivo escrito en el sistema de archivos de un Pod no lo ve ninguna otra réplica**.
2. **Desaparición al recrearse el Pod:** cuando un Pod es destruido por autorreparación (como probamos en [Etapa 6 — Comprobación de autorreparación](README.md#comprobación-de-autorreparación)), por *rolling update* o por *scale-down* del HPA, **su sistema de archivos local desaparece definitivamente** junto con el contenedor, y el reemplazo arranca limpio desde la imagen `jmfloress04/catalogo-api:v1`.

**Cómo lo resuelve `catalogo-app`:**
En [`backend/main.py`](backend/main.py), las imágenes que sube el usuario llegan como *data URI* Base64 (hasta `IMAGEN_MAX_BYTES = 512 KiB`, acotado por debajo del `client_max_body_size 1m` de Nginx) y se almacenan como `bson.Binary` (`imagen_datos`, `imagen_tipo` e `imagen_hash`) **dentro del propio documento en MongoDB**, respaldado por `catalogo-db-pvc` ([Etapa 7](README.md#verificación-de-persistencia-y-cierre-de-la-limitación)), en lugar de escribir archivos en el contenedor de la API. Así, **`catalogo-api` no tiene estado (*stateless*)**: las 3 (o 6) réplicas son 100 % intercambiables y cualquiera puede servir tanto `/productos` como `/productos/{id}/imagen`.

**Prueba experimental (Sexta entrega):** en [Etapa 6](README.md#escalado-a-3-réplicas-y-por-qué-la-api-debe-ser-sin-estado-stateless) y en la **[Sección 5 de INFORME.md](INFORME.md#5-prueba-de-la-sexta-entrega-3-réplicas-de-la-api-y-sistema-de-archivos-aislado-stateless)** escalamos `catalogo-api` a 3 réplicas, escribimos `/tmp/foto.bin` dentro de `catalogo-api-7d4455d6dd-glc9j` y al intentar leerlo desde `catalogo-api-7d4455d6dd-cl5rq` obtuvimos `ls: cannot access '/tmp/foto.bin': No such file or directory`. Si las imágenes del catálogo se hubieran guardado en disco del contenedor, dos de cada tres peticiones habrían fallado al pedir la imagen y un reinicio las habría borrado todas.

**Corolario (carrito en sesión del servidor):** si el catálogo hubiera incorporado un carrito de compras guardado en la sesión en memoria (o disco local) de FastAPI, **tampoco habría sobrevivido al escalar a la segunda réplica**: el cliente agregaría un ítem en la réplica A y, al enviar la siguiente petición balanceada hacia la réplica B, el carrito aparecería vacío o desincronizado (y desaparecería ante cualquier reinicio). Un estado de sesión compartido entre réplicas exige externalizarlo en un almacén centralizado (como Redis o una colección dedicada en MongoDB).

---

## 3. Dificultades propias de MongoDB presentadas durante el curso

Durante las etapas del proyecto se presentaron y resolvieron efectivamente estas **tres dificultades específicas de MongoDB** (todas registradas en [README.md](README.md)):

### 3.1. El parámetro `?authSource=admin` y la decisión de piezas sueltas vs. `MONGO_URI`

> **Etapas de referencia:** [Etapa 3 — Diagnóstico de fallas: resolución de nombre vs. autenticación](README.md#diagnóstico-de-fallas-resolución-de-nombre-vs-autenticación) y [Etapa 7 — Conexión a MongoDB: piezas sueltas vs. `MONGO_URI` completa](README.md#conexión-a-mongodb-piezas-sueltas-vs-mongo_uri-completa).

- **Cómo se manifestó:** al ejecutar `mongosh` en la [Etapa 3](README.md#diagnóstico-de-fallas-resolución-de-nombre-vs-autenticación) contra `"mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogodb"` sin especificar `authSource`, MongoDB rechazó la conexión con `MongoServerError: Authentication failed.`, aun cuando usuario y contraseña eran los correctos.
- **Cómo se diagnosticó:** se contrastó con la prueba inmediata anterior (`DB_HOST=catalogo-bd`), que falló en red/DNS con `[Errno -3] Temporary failure in name resolution`. Al recibir `Authentication failed` desde el propio motor, se confirmó que la red llegaba a MongoDB y que el rechazo se debía a que la imagen oficial `mongo:7` crea el usuario root de `MONGO_INITDB_ROOT_*` en la base **`admin`**, no en `catalogodb`. Sin `?authSource=admin`, MongoDB busca al usuario dentro de `catalogodb` y falla.
- **Cómo se resolvió:** en [Etapa 7](README.md#conexión-a-mongodb-piezas-sueltas-vs-mongo_uri-completa) y [Etapa 8](README.md#4-y-5-chart-de-helm-catalogo-app-12-objetos-exactos-y-parametrización-efectiva) se adoptó el modelo de **piezas sueltas** (`DB_HOST`, `DB_PORT`, `DB_NAME` en [`k8s/configmap-catalogo-api.yaml`](k8s/configmap-catalogo-api.yaml) + `DB_USER` y `DB_PASSWORD` renombradas con `valueFrom.secretKeyRef` desde `catalogo-db-secret` en [`k8s/deploy-catalogo-api.yaml`](k8s/deploy-catalogo-api.yaml)). Así [`backend/main.py`](backend/main.py) construye la URI agregando `?authSource=admin` automáticamente y se evita duplicar la contraseña en el Secret.

### 3.2. Healthcheck con `ping` que responde `healthy` aunque la base no sea utilizable por la API (y variables de inicialización sobre `/data/db` ya poblado)

> **Etapas de referencia:** [Etapa 4 — Healthcheck de `catalogo-db` (MongoDB 7)](README.md#healthcheck-de-catalogo-db-mongodb-7), [Etapa 4 — Healthcheck de `catalogo-api`](README.md#healthcheck-de-catalogo-api-fastapi) y [Etapa 4 — Orden de arranque y dependencias](README.md#orden-de-arranque-y-dependencias-depends_on-con-condition-service_healthy).

- **Cómo se manifestó:** en [`docker-compose.yml`](docker-compose.yml), el `healthcheck` de `catalogo-db` (`mongosh --quiet --eval "db.adminCommand('ping').ok"`) reporta el contenedor como `healthy` incluso si la base arrancó sin autenticación (por omitir `MONGO_INITDB_ROOT_*`) o si las credenciales actuales de `.env` ya no coinciden con las inicializadas previamente en el volumen `catalogo-db-data`.
- **Cómo se diagnosticó:** al analizar el comportamiento de `mongo:7` en [Etapa 4](README.md#healthcheck-de-catalogo-db-mongodb-7), se identificaron dos causas combinadas: (1) `db.adminCommand('ping')` no exige autenticación, por lo que solo comprueba que `mongod` escucha en `:27017`; y (2) el entrypoint de `mongo:7` **solo aplica `MONGO_INITDB_ROOT_*` cuando `/data/db` está vacío**. Si el volumen ya existe, cambiar `.env` no cambia la clave guardada en MongoDB, dando `catalogo-db (healthy)` mientras `catalogo-api` falla al autenticarse.
- **Cómo se resolvió:** se mantuvo `ping` en `catalogo-db` para no incrustar contraseñas en el comando del healthcheck (evitando exponerlas en `docker inspect` o `ps`), pero delegando la comprobación real de usabilidad y credenciales al healthcheck de `catalogo-api` (`/health`, que consulta la base autenticado) y verificando explícitamente la inyección de variables en Kubernetes mediante `kubectl exec deploy/catalogo-api -n catalogo-app -- env` ([Etapa 7](README.md#conexión-a-mongodb-piezas-sueltas-vs-mongo_uri-completa)).

### 3.3. Siembra idempotente (`SEED_DEMO=true`) con múltiples réplicas y prefijo `ROOT_PATH` detrás del proxy/Ingress

> **Etapas de referencia:** [Etapa 3 — Efecto de `ROOT_PATH` en FastAPI](README.md#verificación-del-proxy-y-efecto-de-root_path-en-fastapi), [Etapa 4 — Idempotencia de la siembra de datos](README.md#idempotencia-de-la-siembra-de-datos), [Etapa 7 — Verificación de persistencia](README.md#verificación-de-persistencia-y-cierre-de-la-limitación) y [Etapa 7 — Documentación de la API detrás del Ingress](README.md#documentación-de-la-api-detrás-del-ingress).

- **Cómo se manifestó:** en dos comportamientos propios de la Variante C:
  1. Con `SEED_DEMO=true`, cada reinicio del stack ([Etapa 4](README.md#idempotencia-de-la-siembra-de-datos)) o arranque simultáneo de `2` a `6` réplicas de `catalogo-api` ([Etapas 6 y 8](README.md#3-demostración-bajo-carga-real-2--4--6--3--2)) ejecuta la rutina de siembra sobre la misma colección `productos`; sin idempotencia duplicaría los 12 productos o fallaría por clave duplicada. Además, esos 12 productos pueden inducir un **falso positivo de persistencia** si no se valida con un producto cargado manualmente ([Etapa 7](README.md#verificación-de-persistencia-y-cierre-de-la-limitación)).
  2. Detrás del proxy de Nginx (donde `proxy_pass http://${API_HOST}:${API_PORT}/;` recorta el prefijo `/api`) y luego detrás del Ingress (`http://catalogo.local/api/docs`), FastAPI sin `ROOT_PATH="/api"` generaba un enlace roto a `/openapi.json` en vez de `/api/openapi.json`, dejando Swagger UI en blanco sin errores en logs.
- **Cómo se diagnosticó:** se verificó que tras bajar/subir Compose y tras destruir el Pod de MongoDB la cuenta permaneciera exacta (`12` demo + `1` manual = `13`, nunca `24` ni `25`), y se inspeccionó `/api/docs` y `/api/openapi.json` tras cada etapa de proxy/Ingress (contrastando también `/api/productos` en `application/json` frente al falso positivo SPA `/productos` en `text/html` en [Etapa 7](README.md#exposición-http-con-ingress-catalogo-app-ingress)).
- **Cómo se resolvió:** [`backend/main.py`](backend/main.py) crea un índice único sobre `sku` y siembra con `update_one(..., {"$setOnInsert": doc}, upsert=True)` capturando `DuplicateKeyError`, haciendo segura la concurrencia entre múltiples réplicas; y `ROOT_PATH="/api"` quedó externalizado en [`k8s/configmap-catalogo-api.yaml`](k8s/configmap-catalogo-api.yaml) y [`catalogo-app/values.yaml`](catalogo-app/values.yaml).

---

## 4. Qué quedaría pendiente para un entorno de producción real

El proyecto alcanzado en la [Etapa 8](README.md#etapa-8--escalado-automático-hpa-y-empaquetado-con-helm) es un despliegue completo de laboratorio, pero para operar `catalogo-app` en un entorno de producción real quedarían pendientes cinco pilares:

1. **Observabilidad completa (métricas, logs y trazas):**
   En [Etapa 8](README.md#2-objeto-n-12-creación-del-horizontalpodautoscaler-catalogo-api-hpa) solo disponemos de `metrics-server` (CPU/RAM instantánea para el HPA) y de `kubectl logs`, que se pierden al destruirse un Pod. En producción se necesita:
   - **Métricas históricas y alertas** (Prometheus + Grafana + Alertmanager) sobre latencia p95/p99, tasa de errores 5xx, conexiones de MongoDB y espacio libre del PVC.
   - **Logs centralizados** (Loki o OpenSearch/ELK) en formato JSON estructurado que sobrevivan al reciclado de Pods.
   - **Trazas distribuidas** (OpenTelemetry) extremo a extremo (`Ingress → Nginx → FastAPI → MongoDB`), además de declarar explícitamente `readinessProbe` y `livenessProbe` en los `Deployments` de Kubernetes (llevando al clúster lo que en [Etapa 4](README.md#etapa-4--docker-compose-y-seguridad-básica) se definió como `healthcheck` en Compose).

2. **Integración y despliegue continuos (CI/CD):**
   Durante el curso el build de imágenes ([Etapa 2](README.md#efecto-del-cache-de-construcción)), el escaneo con Trivy ([Etapa 4 — Revisión de seguridad](README.md#revisión-de-seguridad-y-escaneo-de-vulnerabilidades-trivy)) y los `helm upgrade` / `rollback` ([Etapa 8 — Ciclo completo de Release](README.md#7-ciclo-completo-de-release-helm-upgrade-v2-helm-history-y-helm-rollback-v1)) se ejecutaron manualmente. En producción un pipeline automatizado (GitHub Actions / GitLab CI) debe ejecutar tests automáticos, `helm lint`, escaneo Trivy bloqueante ante vulnerabilidades `CRITICAL`/`HIGH` con parche, publicación de imágenes firmadas con tags inmutables por SHA/SemVer, y despliegue continuo vía **GitOps** (Argo CD o Flux).

3. **Gestión externa de secretos y endurecimiento de seguridad:**
   En [Etapa 7 — Secret `catalogo-db-secret`](README.md#secret-catalogo-db-secret-base64-no-es-cifrado) comprobamos que un `Secret` de Kubernetes solo codifica en Base64 (decodificable trivialmente con `base64 -d`) y que en el curso las credenciales de laboratorio se inyectan mediante archivos locales ignorados en `.gitignore`. En producción corresponde:
   - Activar **cifrado en reposo en `etcd`** y controles **RBAC** estrictos.
   - Integrar un **gestor externo de secretos** (HashiCorp Vault, AWS/GCP Secrets Manager o Azure Key Vault) con **External Secrets Operator** (o **Sealed Secrets** / SOPS para GitOps) y rotación periódica de credenciales.
   - Sumar **TLS/HTTPS** en el Ingress (cert-manager + Let's Encrypt), **`NetworkPolicies`** restringiendo el puerto `27017` de `catalogo-db` únicamente a Pods `app: catalogo-api`, y extender el principio de mínimo privilegio de [Etapa 4 (`cap_drop: ALL`)](README.md#principio-de-mínimo-privilegio-cap_drop--cap_add) a `securityContext` en Kubernetes (`runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`).

4. **Copias de respaldo (backups) automatizadas y probadas:**
   El PVC `catalogo-db-pvc` introducido en [Etapa 7](README.md#persistencia-de-mongodb-catalogo-db-pvc) resuelve la efimeridad del Pod, pero **un volumen persistente no es un backup**: no protege frente a borrados accidentales (`DELETE`), corrupción lógica ni fallos del nodo/disco físico donde reside `local-path`. En producción hace falta un `CronJob` o herramienta de respaldo programado (`mongodump --oplog` o snapshots gestionados) almacenado fuera del clúster (almacenamiento de objetos con versionado e inmutabilidad) acompañado de **pruebas periódicas de restauración** para validar RPO y RTO.

5. **MongoDB como Replica Set (por qué un `Deployment` con un PVC no sirve en un entorno serio):**
   Hoy [`k8s/deploy-catalogo-db.yaml`](k8s/deploy-catalogo-db.yaml) ejecuta MongoDB como un `Deployment` de `replicas: 1` con `strategy: Recreate` y un único PVC `ReadWriteOnce` ([Etapa 7](README.md#persistencia-de-mongodb-catalogo-db-pvc)). Esa configuración no es válida para un entorno de producción real por cuatro causas estructurales:
   - **Corte total en cada reinicio o actualización:** `replicas: 1` + `strategy: Recreate` detiene por completo la instancia actual antes de levantar la nueva para evitar colisión sobre el PVC `RWO`, provocando indisponibilidad total en cada mantenimiento o parche.
   - **Punto único de fallo (SPOF) de nodo y disco:** si cae el nodo físico que aloja el volumen `local-path` o se daña el disco, toda la base queda inaccesible sin copia de reemplazo.
   - **Falta de identidad y almacenamiento estable por réplica:** un `Deployment` asume Pods intercambiables compartiendo la misma plantilla de PVC; no puede asignar un nombre DNS estable (`mongo-0`, `mongo-1`, `mongo-2`) ni un volumen dedicado a cada miembro.
   - **Sin replicación ni *failover* automático:** en producción MongoDB debe ejecutarse como un **Replica Set** (mínimo 3 miembros: 1 primario y 2 secundarios sincronizados mediante *oplog*) sobre un **`StatefulSet`** con `volumeClaimTemplates` y un *Headless Service* —idealmente administrado por un **Kubernetes Operator** (Percona / MongoDB Community Operator) o contratado como **servicio gestionado** (p. ej., MongoDB Atlas)— para garantizar elección automática de primario ante fallos, actualizaciones *rolling* sin corte y durabilidad real de los datos.
