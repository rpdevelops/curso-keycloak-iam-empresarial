# ⚛️ Prácticas Sesión 7: Frontend React, Identity Brokering y Keycloakify

En esta sesión vamos a dar cara al usuario final. Aprenderemos cómo una aplicación pública (React) delega su login a Keycloak usando el flujo PKCE. Además, permitiremos a los usuarios iniciar sesión con sus cuentas de Google (Identity Brokering) y personalizaremos completamente la pantalla de login usando Keycloakify.

---

## 🌐 Práctica 1: Configurar el Cliente Público en Keycloak

A diferencia de nuestro Backend Quarkus (que guardaba un "Client Secret"), React es una aplicación pública que se ejecuta en el navegador del usuario y no puede guardar secretos.

### 1.1 Registro del Cliente SPA
1. Ve a la Consola de Keycloak (`Corp-Prod`).
2. Ve a **Clients** y pulsa **Create client**.
3. **Client type:** `OpenID Connect`
4. **Client ID:** `react-frontend`. Pulsa **Next**.
5. En **Capability config**:
   * Client authentication: **OFF** (Vital: es un cliente público).
   * Standard flow: **ON** (Permite el login web con PKCE).
   * Pulsa **Next**.
6. En **Login settings**:
   * **Valid redirect URIs:** `http://localhost:3000/*` (URL de React).
   * **Web origins:** `+` (O `http://localhost:3000` para evitar errores CORS).
7. Pulsa **Save**.

---

## ⚛️ Práctica 2: Integración OIDC en React (Flujo PKCE)

Vamos a crear una aplicación React muy básica y a conectarla con Keycloak usando la librería oficial `keycloak-js`.

### 2.1 Crear el proyecto React
Abre tu terminal y ejecuta:
```bash
npx create-react-app frontend-app
cd frontend-app
npm install keycloak-js
```
### 2.2 Configurar la conexión OIDC
Abre el archivo src/App.js y reemplaza todo su contenido por este código:
```javascript
import React, { useState, useEffect, useRef } from 'react';
import Keycloak from 'keycloak-js';

// Configuración de nuestro Keycloak
const keycloakConfig = {
  url: '[http://keycloak.corp.local](http://keycloak.corp.local)', // URL de NGINX/Keycloak
  realm: 'Corp-Prod',
  clientId: 'react-frontend'
};

const keycloak = new Keycloak(keycloakConfig);

function App() {
  const [authenticated, setAuthenticated] = useState(false);
  const isRun = useRef(false);

  useEffect(() => {
    if (isRun.current) return;
    isRun.current = true;

    // Inicializamos Keycloak con el flujo PKCE automático
    keycloak.init({ onLoad: 'login-required', pkceMethod: 'S256' })
      .then(auth => {
        setAuthenticated(auth);
      })
      .catch(err => console.error("Error al inicializar Keycloak", err));
  }, []);

  if (!authenticated) {
    return <div>Cargando entorno seguro...</div>;
  }

  return (
    <div style={{ padding: '20px' }}>
      <h1>Portal del Empleado</h1>
      <p>Bienvenido, <strong>{keycloak.tokenParsed.preferred_username}</strong>!</p>
      
      <div style={{ margin: '20px 0', padding: '10px', background: '#eee' }}>
        <h3>Tu Token de Acceso (JWT):</h3>
        <textarea readOnly rows="5" style={{ width: '100%' }} value={keycloak.token}></textarea>
      </div>

      <button onClick={() => keycloak.logout()}>Cerrar Sesión</button>
      
      <button onClick={() => keycloak.accountManagement()} style={{ marginLeft: '10px' }}>
        Gestionar mi Cuenta
      </button>
    </div>
  );
}

export default App;
```
### 2.3 Ejecutar y Probar
Ejecuta npm start. La aplicación React te redirigirá automáticamente a la pantalla de login genérica de Keycloak. Haz login con el usuario maria.lopez y verás cómo React recupera el token de forma segura.

## 🤝 Práctica 3: Identity Brokering (Login con Google)
Queremos que los empleados puedan acceder usando su cuenta corporativa de Google Workspace.

(Nota: Para un entorno real necesitas configurar credenciales en Google Cloud Console. Simularemos la configuración en Keycloak).

En la consola de Keycloak, ve a Identity providers en el menú lateral.

Selecciona Google.

Pega el Client ID y Client Secret (te los proporcionaría Google Cloud).

Copia la Redirect URI que te muestra Keycloak (la necesitarás para ponerla en Google).

Pulsa Add.

Vuelve a ejecutar tu app React y verás que mágicamente ha aparecido un botón de "Google" en la pantalla de login. ¡Identity Brokering configurado!

## 🎨 Práctica 4: Branding Corporativo con Keycloakify
Tradicionalmente, personalizar la pantalla de login de Keycloak requería aprender un motor de plantillas Java llamado FreeMarker (HTML/CSS antiguo). Hoy en día, el estándar de la industria para equipos Frontend es usar Keycloakify, que nos permite diseñar el login directamente en React y empaquetarlo para Keycloak.

### 4.1 Inicializar Keycloakify
En tu terminal, dentro de la carpeta frontend-app de React, ejecuta:
```bash
npm install -D keycloakify
npx keycloakify init
```
Esto creará una carpeta src/keycloak-theme con el código fuente del login.

### 4.2 Personalizar la pantalla de Login
Abre el archivo src/keycloak-theme/login/pages/Login.tsx (o el componente equivalente generado).
Busca la cabecera (Header) y añade tu logo corporativo o un mensaje personalizado:
```typescript
// Ejemplo de inyección visual en Login.tsx
<div id="kc-header" className={kcClsx("kcHeaderClass")}>
    <div id="kc-header-wrapper" className={kcClsx("kcHeaderWrapperClass")}>
        <span style={{ fontSize: '2rem', color: '#0056b3' }}>🏢 NeoBank Corp</span>
        <br/>
        {msg("loginAccountTitle")}
    </div>
</div>
```
### 4.3 Compilar el Tema (Build)
Vamos a compilar nuestro código React en un archivo Java (.jar) que Keycloak pueda entender.
```bash
npm run build-keycloak-theme
```
Cuando termine, verás que se ha generado un archivo .jar en la carpeta build_keycloak/target/.

### 4.4 Instalar el Tema en Keycloak
Copia ese archivo .jar (Ej: frontend-app-keycloak-theme-4.1.4.jar) y ponlo en la carpeta local que mapeamos con Ansible en la Sesión 1: /tmp/keycloak-corp/providers/.

Reinicia tu contenedor de Keycloak para que cargue el nuevo proveedor.

En la Consola de Keycloak, ve a Realm settings -> Pestaña Themes.

En el campo Login theme, despliega y selecciona tu nuevo tema React. ¡Pulsa Save!

Entra a tu app React (http://localhost:3000), pulsa Login y admira tu nueva pantalla corporativa.

## 📋 Reto Extra: Declarative User Profile
En versiones recientes, Keycloak introdujo el "User Profile", que permite definir campos de registro dinámicamente sin tocar código frontend. Y lo mejor: ¡Keycloakify lo renderiza automáticamente!

En Keycloak, ve a Realm settings -> Pestaña User Profile.

Pulsa Create attribute.

Name: dni.

En Required, actívalo para user y admin.

En Permissions, marca que el usuario puede verlo y editarlo.

Pulsa Save.

Ahora, cierra sesión en tu App React e intenta registrarte como un usuario nuevo.

Verás que en la pantalla de registro de tu tema Keycloakify, ha aparecido mágicamente un nuevo campo obligatorio llamado "dni", sin que hayas tenido que programar ni un solo `<input>` en React.

***

### Notas sobre la Sesión 7:
1. **Flujo PKCE:** Es crucial recordar que el `Client Secret` nunca, bajo ninguna circunstancia, se pone en una app React, Angular o Vue. PKCE es la solución moderna para SPAs.

¡Con esto cerramos el **Bloque de Desarrollo**! Hemos conseguido conectar el Frontend (React) de forma segura, delegar el login e integrarlo visualmente.