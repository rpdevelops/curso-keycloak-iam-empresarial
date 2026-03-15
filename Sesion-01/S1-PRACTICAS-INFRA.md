# 🛠️ Prácticas Sesión 1: Fundamentos y Despliegue Base (Infraestructura)

En esta primera sesión pondremos en marcha Keycloak por primera vez para entender su funcionamiento básico. Posteriormente, prepararemos nuestro entorno usando Ansible y crearemos una imagen Docker corporativa y estandarizada basada en la distribución Quarkus (LTS), lista para ser desplegada en entornos Kubernetes.

---

## 🚀 Práctica 1: Despliegue Inicial "Hello World" (Modo Dev)

El objetivo de esta práctica es levantar una instancia efímera de Keycloak para familiarizarnos con su interfaz de administración y comprobar que nuestro motor de contenedores funciona correctamente.

### 1.1 Ejecución del contenedor
Abre tu terminal y ejecuta el siguiente comando. Usaremos la versión LTS actual basada en Quarkus:

```bash
docker run --name keycloak-dev -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:24.0.1 start-dev
```

Nota: El flag start-dev habilita un entorno de desarrollo rápido: usa una base de datos en memoria (H2), HTTP en lugar de HTTPS y configuración simplificada.

### 1.2 Acceso y exploración
Abre tu navegador y accede a: http://localhost:8080

Haz clic en "Administration Console".

Inicia sesión con las credenciales que pasamos por variable de entorno (admin / admin).

Explora el menú lateral: observa dónde se gestionan los Realms, Clients, Roles y Users.

Una vez finalizada la exploración, vuelve a la terminal y detén el contenedor pulsando Ctrl + C.

Limpia el entorno ejecutando: docker rm keycloak-dev

## ⚙️ Práctica 2: Preparación del Entorno con Ansible
En un entorno corporativo, la infraestructura se gestiona como código (IaC). Usaremos Ansible para crear la estructura de directorios necesaria en nuestro servidor host (tu máquina) donde alojaremos certificados, temas personalizados y configuraciones que luego inyectaremos en la imagen.

### 2.1 Creación del inventario y playbook
Crea un directorio para esta sesión y entra en él:
```bash
mkdir -p ~/keycloak-curso/sesion1/ansible
cd ~/keycloak-curso/sesion1/ansible
```
Crea un archivo llamado inventory.ini:
```bash
[local]
localhost ansible_connection=local
```
Crea el archivo setup-env.yml (Playbook de Ansible):
```bash
---
- name: Preparar entorno base para Keycloak On-Premise
  hosts: local
  tasks:
    - name: Crear estructura de directorios corporativa
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /tmp/keycloak-corp/certs
        - /tmp/keycloak-corp/themes
        - /tmp/keycloak-corp/providers

    - name: Generar certificado autofirmado (Simulación entorno TLS)
      command: >
        openssl req -x509 -newkey rsa:4096 -keyout /tmp/keycloak-corp/certs/server.key 
        -out /tmp/keycloak-corp/certs/server.crt -days 365 -nodes -subj "/CN=keycloak.corp.local"
      args:
        creates: /tmp/keycloak-corp/certs/server.crt
```
### 2.2 Ejecución del Playbook
Ejecuta el playbook para aprovisionar tu máquina local:
```bash
ansible-playbook -i inventory.ini setup-env.yml
```
Verifica que los directorios y el certificado se han creado correctamente:
```bash
ls -l /tmp/keycloak-corp/certs/
```
## 📦 Práctica 3: Creación de la Imagen Estandarizada (Quarkus)
No debemos usar la imagen genérica de Keycloak directamente en producción. La distribución Quarkus de Keycloak requiere un paso de build (compilación) para optimizar la imagen final, incluir el driver de la base de datos (PostgreSQL), habilitar métricas (Health/Prometheus) y fijar configuraciones.

### 3.1 Creación del Dockerfile
Sal del directorio de Ansible y ve a la raíz de la sesión:
```bash
cd ~/keycloak-curso/sesion1
```
Crea un archivo llamado Dockerfile con el siguiente contenido (utilizamos un patrón multi-stage build):
```bash
# Etapa 1: Build (Optimización de Quarkus)
FROM quay.io/keycloak/keycloak:24.0.1 as builder

# Habilitar métricas y health checks
ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true

# Pre-configurar el proveedor de base de datos
ENV KC_DB=postgres

# Realizar la compilación optimizada
RUN /opt/keycloak/bin/kc.sh build

# Etapa 2: Imagen Final (Estandarizada)
FROM quay.io/keycloak/keycloak:24.0.1
COPY --from=builder /opt/keycloak/ /opt/keycloak/

# Copiar certificados generados por Ansible a la imagen
COPY /tmp/keycloak-corp/certs/server.crt /opt/keycloak/conf/server.crt
COPY /tmp/keycloak-corp/certs/server.key /opt/keycloak/conf/server.key

# Punto de entrada estandarizado para entornos productivos
ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
```
### 3.2 Construcción de la Imagen (Build)
Construye la imagen corporativa etiquetándola apropiadamente:
```bash
docker build -t corp-keycloak:1.0.0-lts .
```
Si ejecutas docker images, deberías ver corp-keycloak en tu lista local. Esta es la imagen inmutable que usaremos en las siguientes sesiones para desplegar en RKE2.
## 🏆 Reto Extra: Simulación de Pipeline (Local Registry)
En una empresa real, tras compilar la imagen (docker build), un pipeline de CI/CD sube esta imagen a un registro privado (Harbor, AWS ECR, Nexus, etc.) para que los clústeres de Kubernetes puedan descargarla. Vamos a simular esto con un registro Docker local.

### Paso 1: Levantar un Docker Registry local
```bash
docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
```
### Paso 2: Etiquetar (Tag) nuestra imagen corporativa
Debemos renombrar la imagen para indicar que pertenece a nuestro registro local (puerto 5000).
```bash
docker tag corp-keycloak:1.0.0-lts localhost:5000/corp-keycloak:1.0.0-lts
```
### Paso 3: Subir (Push) la imagen al registro
```bash
docker push localhost:5000/corp-keycloak:1.0.0-lts
```
### Paso 4: Verificación
Podemos consultar el catálogo de nuestro registro privado consumiendo su API REST con el comando curl:
```bash
curl -X GET http://localhost:5000/v2/_catalog
```
El output debería mostrar: {"repositories":["corp-keycloak"]}

***

### Notas:
* La generación de un **certificado SSL** con `openssl` usando Ansible es crucial para los despliegues de infraestructura On-Premise reales y nos servirá en las siguientes sesiones.
* El `Dockerfile` utiliza el concepto de **Multi-stage build** (etapa `builder`), que es el estándar oficial exigido por Red Hat para la distribución Quarkus de Keycloak en producción.