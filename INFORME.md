# Informe de Funcionamiento — `catalogo-app` (Variante C)

Este informe documenta con registros reales de comandos y sus salidas que el despliegue completo de **`catalogo-app`** (React + Vite servido por Nginx · Python + FastAPI · MongoDB 7) funciona de extremo a extremo sobre Kubernetes y Helm.

## Índice de demostraciones

1. [Aplicación respondiendo a través del Ingress (con producto cargado, imagen y persistencia)](#1-aplicación-respondiendo-a-través-del-ingress-con-producto-cargado-imagen-y-persistencia)
2. [Escalado automático bajo carga (`kubectl get hpa -w`)](#2-escalado-automático-bajo-carga-kubectl-get-hpa--w)
3. [Ciclo completo de Release (`helm upgrade`, `helm rollback` y `helm history`)](#3-ciclo-completo-de-release-helm-upgrade-helm-rollback-y-helm-history)
4. [Parametrización real de `values.yaml` (misma plantilla renderizada dos veces con `helm template`)](#4-parametrización-real-de-valuesyaml-misma-plantilla-renderizada-dos-veces-con-helm-template)
5. [Prueba de la sexta entrega: 3 réplicas de la API y sistema de archivos aislado (*stateless*)](#5-prueba-de-la-sexta-entrega-3-réplicas-de-la-api-y-sistema-de-archivos-aislado-stateless)

---

## 1. Aplicación respondiendo a través del Ingress (con producto cargado, imagen y persistencia)

### 1.1. Despliegue limpio de los 12 objetos con Helm

Instalación de la release sobre un namespace limpio y verificación de todos los recursos creados:

```bash
$ helm install catalogo-app ./catalogo-app \
    -n catalogo-app --create-namespace \
    -f catalogo-app/values-secret.yaml
```

```text
$ kubectl get all,cm,secret,pvc,ingress,hpa -n catalogo-app
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

**Qué se ve:** los 12 objetos del proyecto (3 Deployments, 3 Services, 2 ConfigMaps, 1 Secret, 1 PVC en estado `Bound`, 1 Ingress con `ADDRESS 172.16.21.209` y 1 HPA leyendo métricas reales `cpu: 3%/50%`) levantan con `0` reinicios. Además aparecen los dos objetos internos del clúster/Helm (`kube-root-ca.crt` y `sh.helm.release.v1.catalogo-app.v1`).

### 1.2. Respuesta HTTP a través del Ingress (`http://catalogo.local`)

Con `172.16.21.209 catalogo.local` configurado en `/etc/hosts`, el Ingress enruta `/` hacia `catalogo-frontend:8080`, cuyo Nginx interno sirve la SPA React y hace proxy de `/api/` hacia `catalogo-api:8000`:

```text
$ kubectl get ingress -n catalogo-app
NAME                   CLASS   HOSTS            ADDRESS         PORTS   AGE
catalogo-app-ingress   nginx   catalogo.local   172.16.21.209   80      4m43s
```

```text
# Ruta de la API detrás del Ingress -> 200 OK con application/json (FastAPI + MongoDB):
$ curl -s -i http://catalogo.local/api/productos | head -c 400
HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: application/json
Content-Length: 4297

[{"sku":999666,"categoria":"Apple",...

# Contraste con falso positivo SPA (/productos sin /api) -> 200 OK pero text/html (try_files):
$ curl -s -i http://catalogo.local/productos | head -c 400
HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: text/html
Content-Length: 397

<!doctype html>
<html lang="es">...
```

**Qué se ve:** se distingue el tráfico real que atraviesa toda la cadena (`Ingress → Service frontend → Nginx → Service API → FastAPI → MongoDB`, devolviendo `Content-Type: application/json`) del *fallback* de la SPA React (`try_files` en `/productos`, que devuelve `200 OK` pero con `text/html`). Asimismo, `http://catalogo.local/api/docs` carga Swagger UI apuntando a `/api/openapi.json`, confirmando la propagación de `ROOT_PATH="/api"` desde `catalogo-api-config`.

### 1.3. Producto cargado desde el navegador con su imagen y persistencia en `catalogo-db-pvc`

Desde el navegador en `http://catalogo.local` se cargó un producto manual adicional con su imagen (pasando de los `12` productos sembrados por demo a **`13`** productos totales), y luego se eliminó el Pod de MongoDB para verificar la persistencia del producto y su imagen en el volumen `catalogo-db-pvc`:

```text
# Antes de eliminar el Pod de MongoDB (12 productos demo + 1 cargado manualmente con su imagen):
$ curl -s http://172.16.21.16:30302/api/productos | grep -o '"sku"' | wc -l
13

# Eliminación y recreación automática del Pod de la base de datos:
$ kubectl delete pod -l app=catalogo-db -n catalogo-app
pod "catalogo-db-558886cdd5-5hrjt" deleted from catalogo-app namespace

$ kubectl get pods -n catalogo-app -l app=catalogo-db -w
NAME                           READY   STATUS    RESTARTS   AGE
catalogo-db-558886cdd5-7lf2j   1/1     Running   0          11s

# Después de recrearse el Pod (el producto cargado desde el navegador y su imagen permanecen):
$ curl -s http://172.16.21.16:30302/api/productos | grep -o '"sku"' | wc -l
13
```

**Qué se ve:** los 12 productos demo reaparecerían incluso sin persistencia debido a la siembra idempotente (`SEED_DEMO=true`), por lo que el conteo **`13` antes y después de destruir el Pod de MongoDB** —y la visualización del producto y su imagen en el navegador sobre `http://catalogo.local`— demuestra que tanto el documento como los bytes binarios de la imagen se guardaron en `catalogo-db-pvc` (`/data/db`) y sobrevivieron a la recreación del Pod.

---

## 2. Escalado automático bajo carga (`kubectl get hpa -w`)

Se generó carga sostenida dentro del clúster con 3 Pods temporales (`carga-1`, `carga-2`, `carga-3`) consultando en bucle el endpoint de imágenes (`http://catalogo-api:8000/productos/6b046e42a1573dba71b9c69d/imagen`), lo que obliga a FastAPI a leer los bytes de la imagen desde MongoDB y entregarlos por HTTP. Luego se eliminaron los Pods de carga y se esperó la ventana de estabilización de *scale-down*.

```text
$ kubectl get hpa catalogo-api-hpa -n catalogo-app -w
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

Eventos correspondientes registrados por el controlador HPA:

```text
$ kubectl describe hpa catalogo-api-hpa -n catalogo-app | tail -8
Events:
  Type    Reason             Age   From                       Message
  ----    Reason             Age   From                       Message
  Normal  SuccessfulRescale  10m   horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  10m   horizontal-pod-autoscaler  New size: 6; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  2m1s  horizontal-pod-autoscaler  New size: 3; reason: All metrics below target
  Normal  SuccessfulRescale  106s  horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
```

**Qué se ve:**
- En reposo (`5m9s`), las 2 réplicas consumen `2 %` respecto de `requests.cpu: 100m` (`2m` de CPU), sin estado `<unknown>`.
- Al inyectar carga (`6m45s`), la CPU sube a `208 %/50 %` y el HPA escala rápidamente en 30 segundos siguiendo la secuencia **`2 → 4 → 6`**, deteniéndose exactamente en `maxReplicas: 6` incluso bajo carga sostenida de `120–123 %` para proteger al nodo.
- Al retirar la carga en el minuto `10m` (`cpu: 2%/50%`), el HPA espera la ventana de estabilización de **5 minutos** (hasta el minuto `15m`) para evitar oscilaciones (*flapping*) y luego reduce de forma escalonada **`6 → 3 → 2`**, respetando el piso `minReplicas: 2`.

---

## 3. Ciclo completo de Release (`helm upgrade`, `helm rollback` y `helm history`)

Partiendo de la revisión `1` instalada en el punto 1 (`catalogo-api:v1`, `catalogo-frontend:v1`), se actualizó únicamente el tag de la API a `v2` (repitiendo `-f catalogo-app/values-secret.yaml`, ya que `helm upgrade` no conserva overrides previos automáticamente) y posteriormente se revirtió a la revisión `1`:

### 3.1. Actualización a `v2` (Revisión `2`) y verificación de ciclos de vida independientes

```bash
$ helm upgrade catalogo-app ./catalogo-app \
    -n catalogo-app \
    -f catalogo-app/values-secret.yaml \
    --set api.image.tag=v2
```

```text
$ kubectl get deployments -n catalogo-app -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image'
NAME                IMAGE
catalogo-api        jmfloress04/catalogo-api:v2
catalogo-db         mongo:7
catalogo-frontend   jmfloress04/catalogo-frontend:v1
```

### 3.2. Reversión a la Revisión `1` y consulta del historial

```bash
$ helm rollback catalogo-app 1 -n catalogo-app
```

```text
$ helm history catalogo-app -n catalogo-app
REVISION  UPDATED                   STATUS      CHART               APP VERSION  DESCRIPTION
1         Mon Nov 23 19:35:19 2026  superseded  catalogo-app-0.1.0  v1           Install complete
2         Mon Nov 23 19:41:02 2026  superseded  catalogo-app-0.1.0  v1           Upgrade complete
3         Mon Nov 23 19:42:40 2026  deployed    catalogo-app-0.1.0  v1           Rollback to 1
```

**Qué se ve:**
- En el `upgrade`, cambiar `api.image.tag=v2` actualiza únicamente `catalogo-api` mientras `catalogo-frontend` permanece en `v1` (imágenes y ciclos de despliegue independientes).
- El `rollback` **no elimina la revisión `2`**: crea una nueva **revisión `3`** con el estado completo de la revisión `1` (análogo a `git revert`), devolviendo `catalogo-api` a `v1` y gestionando los 12 objetos como una única unidad transaccional.

---

## 4. Parametrización real de `values.yaml` (misma plantilla renderizada dos veces con `helm template`)

Primero se verifica la validez del chart y que genera exactamente los **12 objetos** esperados:

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
```

A continuación se renderiza la **misma plantilla** (`templates/deploy-catalogo-api.yaml`) dos veces con `helm template` usando valores distintos para demostrar que los parámetros de `values.yaml` son efectivos y no decorativos:

### 4.1. Render 1 — Valores por defecto de `values.yaml` (`replicas: 2`, `tag: v1`)

```text
$ helm template catalogo-app ./catalogo-app -n catalogo-app -s templates/deploy-catalogo-api.yaml
---
# Source: catalogo-app/templates/deploy-catalogo-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-api
  namespace: catalogo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalogo-api
  template:
    metadata:
      labels:
        app: catalogo-api
    spec:
      containers:
        - name: catalogo-api
          image: "jmfloress04/catalogo-api:v1"
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "128Mi"
          envFrom:
            - configMapRef:
                name: catalogo-api-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: catalogo-db-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalogo-db-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
```

### 4.2. Render 2 — Sobrescribiendo con `--set api.image.tag=v2 --set api.replicaCount=5`

```text
$ helm template catalogo-app ./catalogo-app -n catalogo-app -s templates/deploy-catalogo-api.yaml --set api.image.tag=v2 --set api.replicaCount=5
---
# Source: catalogo-app/templates/deploy-catalogo-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-api
  namespace: catalogo-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: catalogo-api
  template:
    metadata:
      labels:
        app: catalogo-api
    spec:
      containers:
        - name: catalogo-api
          image: "jmfloress04/catalogo-api:v2"
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "128Mi"
          envFrom:
            - configMapRef:
                name: catalogo-api-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: catalogo-db-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalogo-db-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
```

**Qué se ve:** al comparar ambos manifiestos generados por la misma plantilla, `spec.replicas` cambia de **`2` a `5`** y `image` cambia de **`"jmfloress04/catalogo-api:v1"` a `"jmfloress04/catalogo-api:v2"`**, probando que los valores de `catalogo-app/values.yaml` inyectan realmente los campos del manifiesto final.

---

## 5. Prueba de la sexta entrega: 3 réplicas de la API y sistema de archivos aislado (*stateless*)

Esta demostración acompaña la **Pregunta 2 de [ARQUITECTURA.md](ARQUITECTURA.md)** (Variante C).

### 5.1. Escalado de `catalogo-api` a 3 réplicas y verificación de respuestas

```bash
$ kubectl scale deployment catalogo-api --replicas=3 -n catalogo-app
```

```text
$ kubectl get pods -n catalogo-app -l app=catalogo-api
NAME                            READY   STATUS    RESTARTS   AGE
catalogo-api-7d4455d6dd-5h8pj   1/1     Running   0          37s
catalogo-api-7d4455d6dd-cl5rq   1/1     Running   0          10m
catalogo-api-7d4455d6dd-glc9j   1/1     Running   0          10m
```

```text
$ for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " http://172.16.21.16:30502/productos; done; echo
200 200 200 200 200 200 200 200 200 200
```

### 5.2. Escritura de un archivo en un Pod y lectura desde otra réplica

```text
# Escritura en la réplica catalogo-api-7d4455d6dd-glc9j:
$ kubectl exec catalogo-api-7d4455d6dd-glc9j -n catalogo-app -- sh -c 'hostname && echo "imagen" > /tmp/foto.bin && ls -l /tmp/foto.bin'
catalogo-api-7d4455d6dd-glc9j
-rw-r--r-- 1 appuser appuser 7 Nov 23 15:07 /tmp/foto.bin

# Intento de lectura desde otra réplica (catalogo-api-7d4455d6dd-cl5rq):
$ kubectl exec catalogo-api-7d4455d6dd-cl5rq -n catalogo-app -- ls -l /tmp/foto.bin
ls: cannot access '/tmp/foto.bin': No such file or directory
command terminated with exit code 2
```

**Qué se ve:** cada réplica de `catalogo-api` tiene su propio sistema de archivos efímero y aislado. El archivo `/tmp/foto.bin` creado en `catalogo-api-7d4455d6dd-glc9j` **no existe** en `catalogo-api-7d4455d6dd-cl5rq` (y desaparecería también si el primer Pod fuera recreado). Esto demuestra empíricamente por qué la API debe ser *stateless* y por qué en `catalogo-app` las imágenes de los productos se persisten dentro de los documentos de MongoDB (`catalogo-db-pvc`) y nunca en el disco local del contenedor.
