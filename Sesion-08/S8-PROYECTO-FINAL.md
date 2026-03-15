# 🚀 Prácticas Sesión 8: Proyecto Final Evolutivo (Implementación E2E)

¡Bienvenidos a la última sesión del curso! Hoy pondremos en práctica todos los conocimientos adquiridos construyendo la arquitectura de identidad completa para **NeoBank Corp**, un banco digital que requiere alta seguridad, despliegue en clúster y personalización total.

Trabajaremos en 4 fases, simulando el ciclo de vida real de un proyecto empresarial.

---

## 🏗️ Fase 1: Despliegue de Infraestructura Base (Equipo DevOps)

El equipo de infraestructura debe garantizar que Keycloak esté disponible, sea resiliente y monitorizable.

### 1.1 Limpieza y Preparación del Clúster
Vamos a asegurarnos de que partimos de un entorno limpio en nuestro clúster RKE2 local:
```bash
kubectl delete namespace iam --ignore-not-found
kubectl create namespace neobank-iam
```
### 1.2 Despliegue de la Imagen Estandarizada
Usaremos la imagen corporativa (corp-keycloak:1.0.1-lts) que construimos con el Dockerfile optimizado para Quarkus en la Sesión 1.

Aplica los manifiestos base (Base de datos PostgreSQL LTS y Keycloak en HA):
```bash
# Navega a la carpeta de manifiestos del proyecto final
cd ~/keycloak-curso/sesion8/manifests

# Desplegar Base de Datos y Secretos
kubectl apply -f 01-postgres-neobank.yaml -n neobank-iam

# Desplegar Keycloak HA (2 Réplicas) y Servicios Headless para Infinispan
kubectl apply -f 02-keycloak-ha.yaml -n neobank-iam

# Desplegar NGINX Ingress
kubectl apply -f 03-ingress-neobank.yaml -n neobank-iam
```
### 1.3 Verificación de Salud y Métricas
Comprueba que los pods están corriendo y que la caché distribuida (JGroups) ha formado el clúster:
```bash
kubectl get pods -n neobank-iam -w
kubectl logs statefulset/keycloak -n neobank-iam | grep "ISPN"
```
(El clúster está listo para recibir la configuración de negocio).
## 👥 Fase 2: Configuración de Negocio y Seguridad (Equipo Usuarios)
Con la infraestructura lista en http://sso.neobank.local, el equipo de Gestión de Identidades asume el control para modelar la organización.

### 2.1 Creación del Realm y Clientes
1. Accede a la Consola de Administración Global (master).
2. Crea el Realm productivo: NeoBank-Prod.
3. Crea el Cliente Frontend (React):
4. Client ID: neobank-web
5. Client type: OpenID Connect
6. Client authentication: OFF (Cliente Público)
7. Valid redirect URIs: http://localhost:3000/*
8. Crea el Cliente Backend (Quarkus):
9. Client ID: neobank-api
10. Client authentication: ON (Confidencial / Service Account)

### 2.2 Modelado RBAC y Atributos
1. Roles Globales: Crea los roles cajero-base y gerente-sucursal.
2. Grupos: Crea el grupo Oficina-Central.
3. Mapeo: Asigna el rol gerente-sucursal al grupo Oficina-Central.
4. Usuarios: Crea el usuario carlos.director y añádelo al grupo Oficina-Central.
5. Atributos: Ve a la pestaña Attributes de Carlos y añádele nivel_autorizacion: ALTO.
6. Mappers: En el cliente neobank-web, crea un mapper tipo "User Attribute" para inyectar nivel_autorizacion en el token JWT.

### 2.3 Flujos de Autenticación y MFA Obligatorio
1.NeoBank exige que los gerentes usen siempre el móvil para acceder.
2. Duplica el flujo Browser y llámalo NeoBank-Browser.
3. Añade una condición de ejecución: Condition - User Role.
4. Configúrala con el alias Es-Gerente y selecciona el rol gerente-sucursal.
5. Debajo de la condición, marca el paso OTP Form como Required.
6. Asigna (Bind) este flujo como el nuevo Browser flow por defecto del Realm.

## 💻 Fase 3: Integración de Aplicaciones (Equipo Desarrollo)
Ahora conectaremos el código de nuestras aplicaciones a la arquitectura IAM de NeoBank.

### 3.1 Backend: API Quarkus Protegida
Abre el proyecto Quarkus de NeoBank. En el archivo application.properties, apunta la seguridad a nuestro nuevo Realm:
```properties
quarkus.oidc.auth-server-url=[http://sso.neobank.local/realms/NeoBank-Prod](http://sso.neobank.local/realms/NeoBank-Prod)
quarkus.oidc.client-id=neobank-api
```
En tu controlador REST (TransaccionesResource.java), protege la ruta crítica exigiendo el rol de gerente y validando el nivel de autorización:
```java
@GET
@Path("/transferencias-alto-valor")
@RolesAllowed("gerente-sucursal")
public Response procesarTransferencia() {
    String nivel = jwt.claim("nivel_autorizacion").orElse("BAJO").toString();
    if (!"ALTO".equals(nivel)) {
        return Response.status(403).entity("Nivel de autorización insuficiente").build();
    }
    return Response.ok("Transferencia de alto valor aprobada").build();
}
```
### 3.2 Frontend: React + Keycloakify
1. Arranca la aplicación React (npm start).
2. Asegúrate de que keycloak-js está apuntando al Realm NeoBank-Prod y al cliente neobank-web.
3. Branding: Compila el tema visual de NeoBank usando Keycloakify (npm run build-keycloak-theme).
4. Sube el .jar resultante al directorio de providers de tu clúster RKE2 (o cópialo dentro del pod) y selecciona el tema "NeoBank" en los ajustes del Realm en Keycloak.

## 🎯 Fase 4: Pruebas End-to-End (Validación Final)
1. Ha llegado el momento de probar que las tres piezas (Infra, Usuarios, Devs) funcionan como un reloj suizo.

2. Intento de Acceso Frontend: Entra a http://localhost:3000. La App React detecta que no tienes sesión y te redirige a Keycloak.

3. Experiencia de Usuario (Branding): Verás la pantalla de inicio de sesión corporativa de NeoBank creada con Keycloakify, no la genérica.

4. Seguridad Adaptativa (MFA): Inicia sesión como carlos.director. Como Carlos hereda el rol gerente-sucursal de su grupo, el motor condicional de Keycloak intercepta el login y exige el Doble Factor (Código QR) de Google Authenticator.

5. Token y Claims: Al completar el login, React recibe el Token JWT. Si lo decodificas en jwt.io, verás el rol gerente-sucursal y el claim personalizado "nivel_autorizacion": "ALTO".

6. Consumo de API: React hace una petición HTTP GET al backend Quarkus (puerto 8081) enviando el Token Bearer.

7. Autorización Backend: Quarkus valida la firma criptográfica, confirma el rol, extrae el claim de autorización y devuelve un 200 OK: Transferencia de alto valor aprobada.

8. Resiliencia (Chaos Test): Como administrador del clúster, borra uno de los pods de Keycloak (kubectl delete pod keycloak-0 -n neobank-iam). Recarga la página de React. Tu sesión sigue activa y válida gracias a la replicación de Infinispan.

***
### ¡Felicidades! Has implementado con éxito una arquitectura IAM Enterprise completa.