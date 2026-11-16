# Informe Técnico: Persistencia y Comunicación Multi-componente en Kubernetes

## 1. Introducción
El presente informe documenta la implementación y verificación de una arquitectura cloud-native multi-componente desplegada sobre **Kubernetes (Minikube)** dentro del namespace `practica-s11`. La solución integra un microservicio reactivo desarrollado con **Spring Boot 3 WebFlux (Java 17)** y **MongoDB 7**, aplicando almacenamiento persistente mediante `PersistentVolumeClaim (PVC)`, externalización de parámetros no sensibles con `ConfigMap`, gestión de credenciales en Base64 mediante `Secret`, resolución de nombres interna por **CoreDNS** (`ClusterIP`) y exposición hacia el exterior mediante `NodePort`.

---

## 2. Explicación Técnica de los Recursos de Kubernetes

### 2.1 PersistentVolumeClaim (`mongo-pvc`)
Los pods en Kubernetes son efímeros: cuando un contenedor se reinicia o es reemplazado por el `Deployment`, cualquier dato escrito en su sistema de archivos interno desaparece. El recurso `PersistentVolumeClaim` (`mongo-pvc`) solicita `1Gi` de almacenamiento con modo de acceso `ReadWriteOnce (RWO)` al provisionador dinámico de Minikube (`standard` / `k8s.io/minikube-hostpath`) y se monta en `/data/db` dentro del contenedor de MongoDB. Además, el `Deployment` de MongoDB utiliza `strategy: Recreate` para asegurar que el pod anterior libere el bloqueo de archivos de WiredTiger antes de que inicie el nuevo pod.

### 2.2 ConfigMap (`app-config`)
De acuerdo con la metodología *12-Factor App*, la configuración de entorno debe estar estrictamente separada del código fuente y de la imagen del contenedor. El `ConfigMap` `app-config` almacena pares clave-valor no sensibles (`DB_HOST=mongo-service`, `DB_PORT=27017`, `DB_NAME=inventario_db`, `APP_PORT=8080`), los cuales se inyectan como variables de entorno en el pod del backend mediante `envFrom.configMapRef`.

### 2.3 Secret (`db-secret`)
Las credenciales de autenticación (`MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`) se almacenan codificadas en Base64 dentro de un objeto `Secret` tipo `Opaque` (`db-secret`). Este mismo `Secret` es consumido tanto por el contenedor `mongo:7` (para inicializar el usuario administrador en `/data/db`) como por el backend Spring WebFlux (para autenticarse contra `authSource=admin`), garantizando consistencia sin exponer contraseñas en el `application.yml` ni en el `Dockerfile`.

### 2.4 Servicios de Red: `ClusterIP` vs `NodePort`
- **`ClusterIP` (`mongo-service`):** Expone el puerto `27017` únicamente dentro de la red virtual interna del clúster (`10.97.41.219`). Gracias a **CoreDNS**, el pod del backend se conecta usando el nombre DNS `mongo-service` sin depender de la IP efímera del pod de MongoDB y sin exponer la base de datos al exterior.
- **`NodePort` (`backend-service`):** Abre el puerto `30080` en el nodo del clúster y redirige el tráfico hacia el puerto `8080` del pod del backend, permitiendo que clientes externos (`curl.exe`, Postman o navegador web) consuman los endpoints reactivos `/api/productos`.

### 2.5 Probes de Salud (`livenessProbe` y `readinessProbe`)
Configuradas sobre los endpoints nativos de **Spring Boot Actuator**:
- **`readinessProbe` (`/actuator/health/readiness`):** Verifica que el servidor Netty haya terminado de arrancar y esté listo para aceptar tráfico. Mientras no responda `200 OK`, el `Service` no le envía peticiones.
- **`livenessProbe` (`/actuator/health/liveness`):** Verifica continuamente que la aplicación no se encuentre bloqueada. Si falla consecutivamente tras el umbral configurado, `kubelet` reinicia automáticamente el contenedor.

---

## 3. Evidencias de Ejecución y Verificación

### Evidencia 1: Salud del Clúster
**Comando ejecutado:**
```powershell
kubectl get all,pvc,configmaps,secrets -n practica-s11
```
**Resultado obtenido (`evidencias/01-estado-cluster.txt`):**
```text
NAME                                      READY   STATUS    RESTARTS   AGE
pod/backend-deployment-7b4b86dc4f-2gv8v   1/1     Running   0          2m
pod/mongo-deployment-59757865cf-t74fn     1/1     Running   0          2m33s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/backend-service   NodePort    10.108.17.5    <none>        8080:30080/TCP   43s
service/mongo-service     ClusterIP   10.97.41.219   <none>        27017/TCP        2m33s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-deployment   1/1     1            1           2m
deployment.apps/mongo-deployment     1/1     1            1           2m33s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-deployment-7b4b86dc4f   1         1         1       2m
replicaset.apps/mongo-deployment-59757865cf     1         1         1       2m33s

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mongo-pvc   Bound    pvc-59300038-ac2d-4cf4-bfc9-41cbfbb0a11e   1Gi        RWO            standard       <unset>                 2m34s

NAME                         DATA   AGE
configmap/app-config         4      2m33s
configmap/kube-root-ca.crt   1      2m34s

NAME               TYPE     DATA   AGE
secret/db-secret   Opaque   2      2m33s
```
**Explicación técnica:**
Se comprueba que todos los recursos declarativos en el namespace `practica-s11` se encuentran operativos. Ambos pods (`backend-deployment-7b4b86dc4f-2gv8v` y `mongo-deployment-59757865cf-t74fn`) están en estado `Running` con disponibilidad `1/1` y `0` reinicios, confirmando la ejecución exitosa de las sondas de salud. Asimismo, se verifica la creación del servicio interno `mongo-service` (`ClusterIP` en `27017/TCP`), el servicio externo `backend-service` (`NodePort` en `8080:30080/TCP`), el `ConfigMap` `app-config`, el `Secret` `db-secret` y el `PersistentVolumeClaim` `mongo-pvc` en estado `Bound`.

---

### Evidencia 2: Persistencia y Enlace del Volumen (`mongo-pvc`)
**Comando ejecutado:**
```powershell
kubectl describe pvc mongo-pvc -n practica-s11
```
**Resultado obtenido (`evidencias/02-describe-pvc.txt`):**
```text
Name:          mongo-pvc
Namespace:     practica-s11
StorageClass:  standard
Status:        Bound
Volume:        pvc-59300038-ac2d-4cf4-bfc9-41cbfbb0a11e
Labels:        app=mongo
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       mongo-deployment-59757865cf-t74fn
Events:
  Normal  ProvisioningSucceeded  2m34s  k8s.io/minikube-hostpath  Successfully provisioned volume pvc-59300038-ac2d-4cf4-bfc9-41cbfbb0a11e
```
**Explicación técnica:**
La inspección de `mongo-pvc` confirma que el provisionador dinámico `k8s.io/minikube-hostpath` asignó el volumen persistente `pvc-59300038-ac2d-4cf4-bfc9-41cbfbb0a11e` de `1Gi` en modo `ReadWriteOnce (RWO)`, alcanzando el estado `Bound`. El campo `Used By: mongo-deployment-59757865cf-t74fn` demuestra que dicho volumen se encuentra montado activamente por el pod de MongoDB en `/data/db`.

---

### Evidencia 3: Inyección de Configuración y Seguridad
**Comando ejecutado:**
```powershell
kubectl exec backend-deployment-7b4b86dc4f-2gv8v -n practica-s11 -- env
```
**Resultado obtenido (`evidencias/03-env-backend.txt`):**
```text
HOSTNAME=backend-deployment-7b4b86dc4f-2gv8v
DB_HOST=mongo-service
DB_NAME=inventario_db
DB_PORT=27017
MONGO_INITDB_ROOT_PASSWORD=K8s$ecret2025!Lab
MONGO_INITDB_ROOT_USERNAME=admin_s11
APP_PORT=8080
HOME=/home/springuser
```
**Explicación técnica:**
Al inspeccionar el entorno de ejecución del contenedor del backend se comprueba que Kubernetes inyectó todas las claves definidas en `app-config` (`DB_HOST`, `DB_NAME`, `DB_PORT`, `APP_PORT`) y decodificó en memoria las credenciales almacenadas en `db-secret` (`MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`). También se verifica que el proceso se ejecuta bajo el usuario sin privilegios `springuser` (`HOME=/home/springuser`).

---

### Evidencia 4: Conectividad Multi-componente y Prueba de Persistencia
**Comandos ejecutados:**
```powershell
# 1. Crear 2 productos (POST) y listarlos (GET)
curl.exe -i -s -X POST "http://127.0.0.1:8080/api/productos" -H "Content-Type: application/json" -d '{\"nombre\": \"Laptop Lenovo ThinkPad P16\", \"precio\": 1850.50}'
curl.exe -i -s -X POST "http://127.0.0.1:8080/api/productos" -H "Content-Type: application/json" -d '{\"nombre\": \"Monitor Dell UltraSharp 27\", \"precio\": 499.99}'
curl.exe -i -s -X GET "http://127.0.0.1:8080/api/productos"

# 2. Destruir el pod de MongoDB y esperar su recreación
kubectl delete pod -l app=mongo -n practica-s11
kubectl wait --for=condition=ready pod -l app=mongo -n practica-s11 --timeout=120s

# 3. Volver a consultar los productos (GET)
curl.exe -i -s -X GET "http://127.0.0.1:8080/api/productos"
```
**Resultado obtenido (`evidencias/04-crud-post-get.txt` y `evidencias/05-prueba-persistencia.txt`):**
```text
=== ANTES DE BORRAR EL POD DE MONGODB ===
Pod de Mongo activo: mongo-deployment-59757865cf-t74fn (IP: 10.244.0.8)
Respuesta GET /api/productos:
HTTP/1.1 200 OK
[{"id":"6afb3ae13449bc40938b46a9","nombre":"Laptop Lenovo ThinkPad P16","precio":1850.5},{"id":"6afb3ae23449bc40938b46aa","nombre":"Monitor Dell UltraSharp 27","precio":499.99}]

=== DESPUÉS DE BORRAR Y RECREAR EL POD DE MONGODB ===
Pod eliminado: mongo-deployment-59757865cf-t74fn
Nuevo Pod creado: mongo-deployment-59757865cf-7f7bz (IP: 10.244.0.10)
Respuesta GET /api/productos:
HTTP/1.1 200 OK
[{"id":"6afb3ae13449bc40938b46a9","nombre":"Laptop Lenovo ThinkPad P16","precio":1850.5},{"id":"6afb3ae23449bc40938b46aa","nombre":"Monitor Dell UltraSharp 27","precio":499.99}]
```
**Explicación técnica:**
La prueba *end-to-end* demuestra que el backend Spring WebFlux recibe peticiones HTTP externas, resuelve el servicio interno `mongo-service` mediante CoreDNS y persiste los documentos en MongoDB (`HTTP/1.1 201 Created`). Al eliminar deliberadamente el pod original de MongoDB (`mongo-deployment-59757865cf-t74fn`, IP `10.244.0.8`), el `Deployment` aprovisionó un pod completamente nuevo (`mongo-deployment-59757865cf-7f7bz`, IP `10.244.0.10`), el cual montó el mismo volumen `mongo-pvc` y fue descubierto transparentemente a través de la IP virtual de `mongo-service`. La consulta `GET /api/productos` posterior retornó exactamente los mismos registros con sus IDs originales, comprobando la persistencia de los datos y la resiliencia de la arquitectura.

---

## 4. Conclusiones
1. **Persistencia Tolerante a Fallos:** El uso de `PersistentVolumeClaim` (`mongo-pvc`) montado en `/data/db` junto con la estrategia `Recreate` garantiza que la destrucción o reinicio del pod de base de datos no ocasione pérdida de información.
2. **Desacoplamiento y Seguridad:** La externalización total de parámetros en `app-config` y credenciales en `db-secret` mediante `envFrom` permite mantener un código fuente y una imagen Docker limpios de datos sensibles.
3. **Aislamiento de Red y Alta Disponibilidad Operativa:** La combinación de `ClusterIP` para proteger la capa de datos internamente, `NodePort` para exponer únicamente la API REST, y sondas `liveness`/`readiness` sobre Spring Boot Actuator asegura una operación estable y auto-recuperable en Kubernetes.
