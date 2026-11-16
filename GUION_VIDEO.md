# Guion de Video de Sustentación (3 a 5 minutos)

**Duración total estimada:** 4 minutos 15 segundos  
**Preparación antes de grabar en Windows:**
1. Tener una ventana de PowerShell en segundo plano ejecutando:
   ```powershell
   kubectl port-forward svc/backend-service -n practica-s11 8080:8080
   ```
   *(O bien `minikube service backend-service -n practica-s11 --url` en una ventana secundaria).*
2. Tener abierto el navegador en:
   - Pestaña 1: `http://localhost:8080/actuator/health`
   - Pestaña 2: `http://localhost:8080/api/productos`
3. Tener la terminal principal de PowerShell ubicada en la raíz del repositorio (`c:\Users\Cristhian\Desktop\Plaintext`).

---

## Estructura Cronometrada por Integrantes

### Bloque 1: Arquitectura y Backend Reactivo (0:00 – 0:50) — Integrante 1
* **En pantalla:** Diagrama de arquitectura en `README.md`, luego mostrar brevemente `src/src/main/resources/application.yml` y `src/Dockerfile`.
* **Guion:**
  > "Buenas tardes profesor. Presentamos nuestra práctica calificada de Kubernetes: *Persistencia y Comunicación Multi-componente* desplegada en el namespace `practica-s11`.
  > Nuestra arquitectura consta de un backend reactivo desarrollado con **Spring Boot 3 WebFlux** y **MongoDB 7**.
  > En `application.yml` comprobamos que la aplicación cumple con el estándar *12-Factor App*: no tiene puertos, IPs ni contraseñas hardcodeadas; construye la URI reactiva de MongoDB exclusivamente con variables de entorno inyectadas por Kubernetes (`APP_PORT`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`), y habilita las sondas de salud `liveness` y `readiness` en `/actuator/health`.
  > En nuestro `Dockerfile` implementamos una construcción *multi-stage*: la primera etapa empaqueta el JAR con Maven y JDK 17, y la segunda etapa ejecuta únicamente el artefacto en una imagen liviana `eclipse-temurin:17-jre-alpine` con un usuario sin privilegios (`springuser`)."

---

### Bloque 2: Manifiestos de Kubernetes (0:50 – 2:00) — Integrante 2
* **En pantalla:** Recorrer los archivos `k8s/01-mongo-pvc.yaml`, `k8s/02-app-configmap.yaml`, `k8s/03-db-secret.yaml`, `k8s/04-mongo-stack.yaml` y `k8s/05-backend-stack.yaml`.
* **Guion:**
  > "Pasando a los manifiestos en la carpeta `k8s/`:
  > 1. En `01-mongo-pvc.yaml` declaramos el `PersistentVolumeClaim` `mongo-pvc` con `1Gi` de almacenamiento y modo de acceso `ReadWriteOnce`.
  > 2. En `02-app-configmap.yaml` definimos los parámetros no sensibles, donde `DB_HOST` apunta al nombre DNS `mongo-service` que resuelve internamente **CoreDNS**.
  > 3. En `03-db-secret.yaml` almacenamos el usuario y contraseña de MongoDB codificados en Base64 dentro de un Secret tipo `Opaque`.
  > 4. En `04-mongo-stack.yaml` configuramos el Deployment de `mongo:7` con estrategia `Recreate` para evitar bloqueos sobre el volumen, montamos `mongo-pvc` en `/data/db` y exponemos la base de datos únicamente dentro del clúster mediante el Service `ClusterIP` `mongo-service` en el puerto `27017`.
  > 5. Finalmente, en `05-backend-stack.yaml` inyectamos tanto el ConfigMap como el Secret mediante `envFrom`, definimos límites de CPU y memoria, configuramos las `livenessProbe` y `readinessProbe` contra Actuator, y exponemos el servicio hacia el exterior mediante `backend-service` de tipo `NodePort` en el puerto `30080`."

---

### Bloque 3: Ejecución en Vivo y Prueba de Persistencia (2:00 – 4:15) — Integrante 3
* **En pantalla:** Terminal PowerShell y navegador web.
* **Comandos a ejecutar en vivo:**
  ```powershell
  # 1. Salud del clúster y estado Bound del PVC
  kubectl get all,pvc,configmaps,secrets -n practica-s11
  kubectl describe pvc mongo-pvc -n practica-s11

  # 2. Inyección de variables en el contenedor del backend
  $BACKEND_POD = kubectl get pods -n practica-s11 -l app=backend -o jsonpath="{.items[0].metadata.name}"
  kubectl exec $BACKEND_POD -n practica-s11 -- env | Select-String "DB_|APP_PORT|MONGO_"

  # 3. Creación y lectura de productos
  curl.exe -i -X POST "http://localhost:8080/api/productos" -H "Content-Type: application/json" -d "{\"nombre\": \"Servidor Dell PowerEdge\", \"precio\": 3500.00}"
  curl.exe -s "http://localhost:8080/api/productos"

  # 4. Prueba de persistencia: eliminar el pod de MongoDB y verificar que los datos sobreviven
  kubectl delete pod -l app=mongo -n practica-s11
  kubectl wait --for=condition=ready pod -l app=mongo -n practica-s11 --timeout=120s
  kubectl get pods -n practica-s11
  curl.exe -s "http://localhost:8080/api/productos"
  ```
* **Guion:**
  > "Ahora validamos el despliegue en vivo desde PowerShell:
  > - Primero ejecutamos `kubectl get all,pvc,configmaps,secrets -n practica-s11`: vemos ambos pods en estado `1/1 Running`, los servicios `ClusterIP` y `NodePort` activos, y `mongo-pvc` en estado `Bound`.
  > - Con `kubectl describe pvc mongo-pvc` verificamos que el volumen está asociado al pod de MongoDB (`Used By`).
  > - Con `kubectl exec` consultamos las variables de entorno dentro del pod del backend y confirmamos que Kubernetes inyectó correctamente los valores del ConfigMap y del Secret.
  > - Ahora realizamos una petición `POST` a `/api/productos` con `curl.exe`, obteniendo `201 Created`, y al refrescar `http://localhost:8080/api/productos` en el navegador vemos los productos almacenados en MongoDB.
  > - Por último, demostramos la persistencia: eliminamos el pod de MongoDB con `kubectl delete pod -l app=mongo -n practica-s11`. Observamos cómo el Deployment crea un pod nuevo con un identificador distinto, y cuando vuelve a estar `Ready`, consultamos nuevamente `/api/productos` en la terminal y en el navegador: todos los productos permanecen intactos gracias al volumen persistente `mongo-pvc`. Muchas gracias."
