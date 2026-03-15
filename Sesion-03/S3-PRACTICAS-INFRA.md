# 🏢 Prácticas Sesión 3: Multi-Tenant, LDAP, Backups y Operaciones Day-2

En esta sesión abordaremos las operaciones críticas del día a día (Day-2 Operations). Un clúster de Keycloak en producción alojado en RKE2 no solo sirve a una aplicación, sino a toda la compañía. Aprenderemos a aislar entornos, conectar directorios corporativos antiguos (LDAP) y garantizar la supervivencia de los datos ante desastres.

---

## 🏗️ Práctica 1: Gestión Multi-Tenant y Delegación

Keycloak permite alojar múltiples "Reinos" (Realms) en una sola instancia. Cada Realm es un inquilino aislado (Multi-Tenant). Vamos a simular la separación de entornos y a crear un administrador delegado que solo tenga permisos sobre un entorno concreto.

### 1.1 Creación de entornos aislados
1. Accede a tu Consola de Administración (`http://keycloak.corp.local`).
2. Despliega el menú superior izquierdo (donde dice `master`) y haz clic en **Create Realm**.
3. Crea un Realm llamado `Corp-QA`.
4. Repite el proceso y crea otro llamado `Corp-Prod`.

### 1.2 Delegación de Administración
No queremos que los desarrolladores de QA puedan borrar el entorno de Producción.
1. Vuelve al Realm **master** (el único realm desde donde se pueden gestionar los demás).
2. Ve a **Users** y crea un usuario llamado `admin-qa`. Ponle una contraseña en la pestaña *Credentials*.
3. Ve a la pestaña **Role Mapping** de `admin-qa`.
4. Haz clic en **Assign role**, y en el buscador filtra por "Filter by clients".
5. Busca el cliente `Corp-QA-realm` y selecciona el rol `realm-admin`.
6. **Validación:** Abre una ventana de incógnito, ve a `http://keycloak.corp.local/admin/Corp-QA/console` e inicia sesión con `admin-qa`. Verás que solo tiene acceso a gestionar ese Realm específico.

---

## 📇 Práctica 2: Integración de Directorio Activo / LDAP

Las grandes corporaciones ya tienen sus identidades en Active Directory o LDAP. Keycloak no necesita importar esas contraseñas, puede "federarlas".

### 2.1 Desplegar un LDAP de prueba en RKE2
Para poder practicar, levantaremos un servidor OpenLDAP de prueba dentro de nuestro clúster.
Crea el archivo `04-openldap.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: openldap
  namespace: iam
spec:
  selector:
    app: openldap
  ports:
    - port: 389
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openldap
  namespace: iam
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openldap
  template:
    metadata:
      labels:
        app: openldap
    spec:
      containers:
      - name: openldap
        image: osixia/openldap:1.5.0
        env:
        - name: LDAP_ORGANISATION
          value: "ImaginaCorp"
        - name: LDAP_DOMAIN
          value: "corp.local"
        - name: LDAP_ADMIN_PASSWORD
          value: "admin123"
        ports:
        - containerPort: 389
```
Aplica el manifiesto: 
```bash
kubectl apply -f 04-openldap.yaml
```

### 2.2 Configurar la Federación (User Federation)
En Keycloak, sitúate en el Realm Corp-Prod.

Ve a User Federation en el menú lateral y añade un proveedor LDAP.

Configura los siguientes parámetros (deja el resto por defecto):

Vendor: Other

Connection URL: ldap://openldap.iam.svc.cluster.local:389 (Usamos el DNS interno de Kubernetes)

Users DN: dc=corp,dc=local

Bind Type: simple

Bind DN: cn=admin,dc=corp,dc=local

Bind Credential: admin123

Pulsa Save y luego haz clic en el botón Synchronize all users.

## 💾 Práctica 3: Estrategias de Backup
Existen dos tipos de backups en Keycloak: el físico (Base de datos) y el lógico (JSON). Veremos cómo hacer ambos en nuestro entorno Kubernetes.

### 3.1 Backup Físico (Dump de PostgreSQL)
Es la copia de seguridad real y obligatoria en producción. Mantiene todas las contraseñas, configuraciones y sesiones activas.

Ejecutaremos pg_dump directamente dentro del pod de PostgreSQL:
```bash
# Crear el volcado (dump)
kubectl exec -it statefulset/postgres -n iam -- pg_dump -U keycloak -d keycloak -F c -f /tmp/keycloak-db-backup.dump

# Extraer el archivo de backup a tu máquina local
kubectl cp iam/postgres-0:/tmp/keycloak-db-backup.dump ./keycloak-db-backup.dump
```
### 3.2 Backup Lógico (Exportación de Realm a JSON)
Se usa para mover configuraciones de un entorno a otro (ej. de QA a Producción) o guardar la infraestructura como código.

En Keycloak, ve a Realm settings -> Botón superior de acciones Action -> Partial export.

Marca la opción Include clients e Include groups and roles.

Haz clic en Export. Se descargará un archivo .json en tu máquina.

## 🔥 Reto Extra: Actualización Zero-Downtime (Rolling Update)
Vamos a simular que Red Hat ha lanzado un parche crítico de seguridad (v1.0.1-lts) y necesitamos actualizar nuestra imagen estandarizada sin tirar el servicio, apoyándonos en la Alta Disponibilidad (HA) que montamos en la Sesión 2.

### Paso 1: Simular la nueva imagen
En tu terminal local, etiqueta la imagen actual con una nueva versión para simular el parche:
```bash
docker tag corp-keycloak:1.0.0-lts corp-keycloak:1.0.1-lts
```
### Paso 2: Aplicar el Rolling Update en K8s
Vamos a decirle a Kubernetes que actualice el StatefulSet a la nueva imagen.
```bash
kubectl set image statefulset/keycloak keycloak=corp-keycloak:1.0.1-lts -n iam
```
### Paso 3: Monitorizar la actualización
Observa cómo Kubernetes apaga un pod (keycloak-1), lo actualiza, espera a que esté sano (gracias a los Readiness Probes), y solo entonces procede a actualizar el otro (keycloak-0):
```bash
kubectl rollout status statefulset/keycloak -n iam
```
Comprobación Crítica: Durante este proceso, si intentas navegar por la consola de Keycloak, no deberías notar ningún corte. NGINX Ingress enruta el tráfico al pod que sigue vivo, y la caché de JGroups sincroniza las sesiones una vez que el nuevo pod se levanta.

***

### Notas:
1. **Backups Nativos K8s:** Utilizar `kubectl exec` y `kubectl cp`. El export lógico de la UI (Partial Export) es la forma recomendada en Quarkus para sacar plantillas, ya que el comando `kc.sh export` por CLI suele requerir detener el servidor o usar herramientas externas.
2. **Rolling Updates:** Este reto extra es la culminación de todo el bloque de Infraestructura. Demuestra por qué usamos K8s (RKE2), Nginx Ingress, StatefulSets y Probes.

¡Con esto cerramos el **Bloque 1: Infraestructura**!