# ☕ Prácticas Sesión 6: Backend Java (Quarkus), APIs y Microservicios

En esta sesión actuaremos como desarrolladores Backend. Nuestro objetivo es crear una API REST moderna con Java y Quarkus que no confíe en nadie por defecto. Esta API validará criptográficamente los tokens JWT emitidos por nuestro Keycloak (Sesiones anteriores), protegerá rutas específicas basándose en los roles del usuario y extraerá datos de negocio (Claims) inyectados en el token.

---

## 🚀 Práctica 1: Creación del Proyecto Quarkus (Maven) y Dependencias

Vamos a generar un proyecto base de Quarkus configurado con Maven y las extensiones necesarias para seguridad OIDC (OpenID Connect) y servicios REST.

### 1.1 Generar el proyecto base
Abre tu terminal, ve a la carpeta de tu código y ejecuta el comando de Maven para crear la estructura (usaremos la versión LTS de Java y Quarkus):

```bash
mvn io.quarkus.platform:quarkus-maven-plugin:3.8.3:create \
    -DprojectGroupId=com.corp \
    -DprojectArtifactId=backend-api \
    -Dextensions="resteasy-reactive-jackson,oidc,smallrye-jwt"
```
(Nota: Si usaste la web code.quarkus.io para generar el ZIP, asegúrate de haber marcado las extensiones de RESTEasy Reactive Jackson y OpenID Connect).

Abre el archivo pom.xml generado y verifica que el bloque de dependencias contiene las librerías vitales:
```xml
<dependencies>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-oidc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-jwt</artifactId>
    </dependency>
    </dependencies>
```
### 1.2 Configurar la conexión a Keycloak
Abre el archivo src/main/resources/application.properties y añade la configuración para que Quarkus sepa dónde ir a validar las firmas de los tokens JWT.
```properties
# Configuración del servidor de Keycloak
quarkus.oidc.auth-server-url=[http://keycloak.corp.local/realms/Corp-Prod](http://keycloak.corp.local/realms/Corp-Prod)
quarkus.oidc.client-id=app-contabilidad

# Nuestra API es un "Resource Server" (Solo acepta tokens Bearer, no hace login web)
quarkus.oidc.application-type=service

# Cambiamos el puerto para que no choque con nuestro Keycloak local
quarkus.http.port=8081

# Opcional: Para desarrollo local, si no usamos HTTPS internamente
quarkus.oidc.tls.verification=none
```
## 🛡️ Práctica 2: Protección de Endpoints (RBAC) y Extracción de Claims
Vamos a crear un controlador REST (Resource en JAX-RS) que exponga tres endpoints con diferentes niveles de seguridad.

### 2.1 Crear el Controlador REST
Crea un archivo llamado FacturasResource.java en src/main/java/com/corp/:
```java
package com.corp;

import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.jwt.JsonWebToken;

import io.quarkus.security.Authenticated;

@Path("/api/facturas")
public class FacturasResource {

    // Inyectamos el Token JWT actual de la petición
    @Inject
    JsonWebToken jwt;

    // 1. Endpoint Público (Sin seguridad)
    @GET
    @Path("/publico")
    @Produces(MediaType.TEXT_PLAIN)
    public String endpointPublico() {
        return "Información pública de la empresa (No requiere Token)";
    }

    // 2. Endpoint Autenticado (Requiere Token válido, cualquier rol)
    @GET
    @Path("/basico")
    @Authenticated
    @Produces(MediaType.TEXT_PLAIN)
    public String endpointBasico() {
        return "Hola " + jwt.getName() + ", tu token es válido.";
    }

    // 3. Endpoint Protegido (Requiere Rol específico y extrae Claims personalizados)
    @GET
    @Path("/vip")
    @RolesAllowed("director-financiero")
    @Produces(MediaType.APPLICATION_JSON)
    public RespuestaVip endpointVip() {
        // Extraemos el claim personalizado que inyectó el equipo de Usuarios en la Sesión 4
        String centroCoste = jwt.claim("empresa_centro_coste").orElse("DESCONOCIDO").toString();
        
        return new RespuestaVip(
            jwt.getName(),
            "Acceso concedido a datos financieros.",
            centroCoste
        );
    }

    // Clase auxiliar para la respuesta JSON
    public static class RespuestaVip {
        public String usuario;
        public String mensaje;
        public String centroCoste;

        public RespuestaVip(String usuario, String mensaje, String centroCoste) {
            this.usuario = usuario;
            this.mensaje = mensaje;
            this.centroCoste = centroCoste;
        }
    }
}
```
### 2.2 Ejecutar la API
Arranca tu aplicación Quarkus en modo desarrollo usando el wrapper de Maven:
```bash
cd backend-api
./mvnw compile quarkus:dev
```
La API estará escuchando en http://localhost:8081.

## 🤖 Práctica 3: Microservicios y Batch Processing (Machine-to-Machine)
Hasta ahora, los tokens representaban a humanos (ej. María). Pero, ¿qué pasa si un proceso nocturno (Batch) o un microservicio interno necesita llamar a nuestra API protegida? No hay humano para poner usuario y contraseña. Usaremos el flujo Client Credentials Grant (Service Accounts).

### 3.1 Configurar una Service Account en Keycloak
Ve a la Consola de Keycloak (Corp-Prod).

Ve a Clients y pulsa Create client.

Client ID: batch-processor.

En Capability config:

Client authentication: ON (Es un cliente confidencial).

Standard flow: OFF (No hace login web).

Service accounts roles: ON (Vital para Machine-to-Machine).

Pulsa Save.

Ve a la pestaña Credentials de batch-processor y copia el Client Secret.

Ve a la pestaña Service account roles y asígnale el rol director-financiero (o el rol que proteja tu API).

### 3.2 Obtener el Token (Simulación del proceso Batch)
Un proceso Batch en Python o Java pediría el token enviando sus credenciales directamente. Vamos a simularlo con curl:
```bash
curl -X POST [http://keycloak.corp.local/realms/Corp-Prod/protocol/openid-connect/token](http://keycloak.corp.local/realms/Corp-Prod/protocol/openid-connect/token) \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=batch-processor" \
  -d "client_secret=AQUI_PEGA_TU_CLIENT_SECRET"
```
El servidor te devolverá un JSON con un access_token.

### 3.3 Consumir la API Quarkus protegida
Copia ese access_token e invoca a tu endpoint protegido de Quarkus:
```bash
curl -X GET http://localhost:8081/api/facturas/vip \
  -H "Authorization: Bearer AQUI_PEGA_EL_ACCESS_TOKEN"
```
Si todo está correcto, Quarkus validará la firma del token y te devolverá el JSON con acceso concedido.

## 🔥 Reto Extra: Testeando Roles Localmente sin Keycloak
En el día a día, un desarrollador backend no quiere tener que levantar todo un clúster de Keycloak para probar sus tests unitarios. Quarkus ofrece OidcWiremock para simular tokens JWT.

Añade a tu clase de Test (FacturasResourceTest.java) el uso de @TestSecurity:
```java
package com.corp;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.quarkus.test.security.oidc.Claim;
import io.quarkus.test.security.oidc.OidcSecurity;
import org.junit.jupiter.api.Test;
import static io.restassured.RestAssured.given;

@QuarkusTest
public class FacturasResourceTest {

    @Test
    @TestSecurity(user = "testUser", roles = "director-financiero")
    @OidcSecurity(claims = {
        @Claim(key = "empresa_centro_coste", value = "TEST-LAB-01")
    })
    public void testEndpointVipConPermisos() {
        given()
          .when().get("/api/facturas/vip")
          .then()
             .statusCode(200);
    }
}
```
Ejecuta el test con Maven:
```bash
./mvnw test
```
¡Tu código valida la seguridad y extrae los claims sin necesidad de hacer peticiones de red a Keycloak!

***
### Notas:
- . **El puerto de Quarkus:** Lo mencioné en el texto, pero estate atento a esto al hacer las pruebas: por defecto Quarkus levanta en el `8080`, igual que nuestro contenedor base de Keycloak si lo corremos local (aunque en RKE2 lo tenemos mapeado por Ingress al puerto 80/443). Cambiar `quarkus.http.port=8081` en desarrollo siempre es una buena práctica.