# ☸️ Prácticas Sesión 2: Despliegue en Kubernetes, HA y Observabilidad

En esta sesión llevaremos la imagen estandarizada de Keycloak que creamos en la Sesión 1 a un entorno de orquestación real. Utilizaremos **RKE2** (la distribución certificada de Rancher) para desplegar nuestra infraestructura utilizando manifiestos YAML puros, garantizando Alta Disponibilidad (HA) y configurando el Proxy Inverso (NGINX Ingress) que viene integrado.

---

## 🏗️ Práctica 1: Preparación del Namespace y Base de Datos (PostgreSQL LTS)

Keycloak requiere una base de datos relacional robusta. Desplegaremos PostgreSQL (versión 15 LTS) y protegeremos sus credenciales utilizando K8s Secrets.

### 1.1 Crear el espacio de trabajo
Abre tu terminal, crea una carpeta para los manifiestos de esta sesión y define el Namespace:

```bash
mkdir -p ~/keycloak-curso/sesion2/manifests
cd ~/keycloak-curso/sesion2/manifests

# Crear el namespace dedicado para Gestión de Identidades
kubectl create namespace iam
```
### 1.2 Crear el Secreto para la Base de Datos
Nunca debemos escribir contraseñas en texto plano en nuestros despliegues. Creamos un secreto opaco:
```bash
kubectl create secret generic keycloak-db-secret \
  --from-literal=POSTGRES_USER=keycloak \
  --from-literal=POSTGRES_PASSWORD=KeycloakCorp2026! \
  --from-literal=POSTGRES_DB=keycloak \
  -n iam
```
### 1.3 Manifiesto de PostgreSQL (StatefulSet + Service)
Crea el archivo 01-postgres.yaml:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: iam
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: iam
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
        - secretRef:
            name: keycloak-db-secret
        ports:
        - containerPort: 5432
        # RKE2 incluye el 'local-path' provisioner por defecto
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```
Aplica el manifiesto y verifica que el pod se levanta correctamente:
```bash
kubectl apply -f 01-postgres.yaml
kubectl get pods -n iam -w
```
(Espera hasta que el pod postgres-0 esté en estado Running antes de continuar).
## 🚀 Práctica 2: Despliegue de Keycloak en Alta Disponibilidad (HA)
Para que Keycloak funcione en clúster (Alta Disponibilidad), sus réplicas necesitan descubrirse entre sí para sincronizar las sesiones en memoria (Infinispan). En Kubernetes, esto se logra mediante el protocolo DNS_PING de JGroups a través de un "Headless Service".

### 2.1 Manifiesto de Keycloak (HA y Configuración)
Crea el archivo 02-keycloak.yaml. Fíjate que usaremos la imagen corp-keycloak:1.0.0-lts que construimos en la Sesión 1 (asegurando la política IfNotPresent para que K8s use la imagen local).
```yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak-discovery # Headless Service para JGroups
  namespace: iam
spec:
  clusterIP: None
  selector:
    app: keycloak
  ports:
    - port: 7800
      name: jgroups
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: iam
spec:
  selector:
    app: keycloak
  ports:
    - port: 8080
      name: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: keycloak
  namespace: iam
spec:
  serviceName: "keycloak-discovery"
  replicas: 2 # Desplegamos 2 nodos para HA
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: corp-keycloak:1.0.0-lts
        imagePullPolicy: IfNotPresent
        args: ["start"] # Usamos 'start' (producción), no 'start-dev'
        env:
        # Credenciales Admin (En prod, usar Secrets)
        - name: KEYCLOAK_ADMIN
          value: "admin"
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: "admin"
        # Configuración de Base de Datos
        - name: KC_DB_URL
          value: "jdbc:postgresql://postgres:5432/keycloak"
        - name: KC_DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: keycloak-db-secret
              key: POSTGRES_USER
        - name: KC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: keycloak-db-secret
              key: POSTGRES_PASSWORD
        # Configuración Clustering (Infinispan / JGroups)
        - name: KC_CACHE_STACK
          value: "kubernetes"
        - name: JGROUPS_DISCOVERY_PROTOCOL
          value: "dns.DNS_PING"
        - name: JGROUPS_DISCOVERY_PROPERTIES
          value: "dns_query=keycloak-discovery.iam.svc.cluster.local"
        # Configuración Proxy
        - name: KC_PROXY
          value: "edge"
        - name: KC_HOSTNAME_STRICT
          value: "false"
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 7800
          name: jgroups
        # Readiness Probe vital para HA
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
```
Aplica el manifiesto:
```bash
kubectl apply -f 02-keycloak.yaml
```
Verifica cómo se van levantando las dos réplicas en paralelo:
```bash
kubectl get pods -n iam -w
```
## 🛡️ Práctica 3: Exposición Externa (NGINX Ingress)
RKE2 viene con NGINX Ingress Controller preinstalado. Vamos a crear una regla de enrutamiento para que el tráfico externo llegue a nuestro servicio de Keycloak.

Crea el archivo 03-ingress.yaml:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak-ingress
  namespace: iam
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
spec:
  ingressClassName: nginx
  rules:
  - host: keycloak.corp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: keycloak
            port:
              number: 8080
```
Aplica el Ingress:
```bash
kubectl apply -f 03-ingress.yaml
```
Configuración local: Para que tu navegador entienda la URL keycloak.corp.local, debes añadirla a tu archivo hosts (/etc/hosts en Linux/Mac o C:\Windows\System32\drivers\etc\hosts en Windows), apuntando a 127.0.0.1.
```bash
127.0.0.1   keycloak.corp.local
```
Accede desde tu navegador a: http://keycloak.corp.local
## 📊 Práctica 4: Observabilidad (Métricas)
En el Dockerfile de la Sesión 1 ya activamos KC_METRICS_ENABLED=true. Esto expone un endpoint en el formato estándar de Prometheus.

Comprobemos que las métricas de negocio y de la Máquina Virtual Java (Quarkus) se están generando correctamente:
```bash
# Hacemos un port-forward temporal a una de las réplicas
kubectl port-forward pods/keycloak-0 8080:8080 -n iam
```
Abre otra terminal o el navegador y accede a:
```bash
curl http://localhost:8080/metrics
```
Deberías ver una larga lista de métricas puras, incluyendo vendor_cache_manager_keycloak_cache_ (Métricas de Infinispan) y base_memory_usedHeap_bytes.
## 🔥 Reto Extra: Chaos Engineering y Caché Distribuida
Vamos a comprobar si la Alta Disponibilidad (HA) que hemos configurado funciona realmente. El clúster de JGroups debería estar replicando los inicios de sesión entre keycloak-0 y keycloak-1.

### Paso 1: Comprobar el clúster JGroups
Mira los logs del primer pod de Keycloak y busca la confirmación de que ha encontrado al segundo pod:
```bash
kubectl logs keycloak-0 -n iam | grep "ISPN"
```
Deberías ver un mensaje similar a: ISPN000094: Received new cluster view... [keycloak-0|1] (2) [keycloak-0, keycloak-1].

### Paso 2: Crear una sesión activa
Abre tu navegador y accede a la Consola de Administración (http://keycloak.corp.local).

Inicia sesión con admin / admin.

### Paso 3: Destruir un nodo (Chaos)
Mientras estás logueado en la consola web, destruye brutalmente uno de los pods:
```bash
kubectl delete pod keycloak-0 -n iam
```
### Paso 4: Comprobar la resiliencia
Vuelve rápidamente a tu navegador y recarga la página o navega por los menús de la Consola de Administración.
Resultado esperado: ¡Sigues logueado! NGINX Ingress ha redirigido tu tráfico transparente a keycloak-1, y gracias a Infinispan, este segundo nodo ya tenía una copia exacta de tu sesión en memoria.

***
### Notas:
**DNS_PING:** Esta es la configuración clave en el entorno Quarkus de Keycloak en K8s. La variable `JGROUPS_DISCOVERY_PROPERTIES=dns_query=keycloak-discovery...` hace que los nodos se encuentren a través del Headless Service.
**Ingress RKE2:** RKE2 usa NGINX por defecto, por lo que el `ingressClassName: nginx` funcionará sin necesidad de instalar controladores extra.