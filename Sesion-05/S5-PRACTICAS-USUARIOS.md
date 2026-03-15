# 🛡️ Prácticas Sesión 5: Autenticación Avanzada (MFA), Flujos y Políticas (ABAC)

En esta sesión tomaremos el control absoluto de cómo los usuarios inician sesión y a qué recursos tienen acceso. Aprenderemos a personalizar el flujo de autenticación para exigir un Doble Factor (OTP), crearemos reglas condicionales y configuraremos el motor de Autorización de Keycloak para proteger recursos basándonos en atributos y no solo en roles.

---

## 📱 Práctica 1: Flujos de Autenticación y Doble Factor (2FA)

Por defecto, Keycloak solo pide usuario y contraseña (Browser Flow). Vamos a crear un flujo corporativo personalizado para obligar a los empleados a usar una app de autenticación (Google Authenticator, FreeOTP).

### 1.1 Duplicar el Flujo Base
Nunca modifiques los flujos por defecto de Keycloak. Siempre haz una copia.
1. Ve a **Authentication** en el menú lateral.
2. Selecciona el flujo **browser** (que es el que se usa vía web).
3. Haz clic en el botón superior derecho (tres puntos) o **Action** -> **Duplicate**.
4. Nombra el nuevo flujo como `browser-corp` y pulsa **Duplicate**.

### 1.2 Configurar OTP Obligatorio
1. En tu nuevo flujo `browser-corp`, verás una estructura de árbol.
2. Busca el paso llamado **Browser OTP Form** (suele estar dentro de *Browser - Conditional OTP*).
3. A la derecha de ese paso, verás que su requisito está en `Conditional` o `Disabled`.
4. Cámbialo a **Required** (Obligatorio).

### 1.3 Aplicar el nuevo Flujo al Realm
1. Vuelve a la pestaña principal de **Authentication** o pulsa sobre las acciones de tu flujo `browser-corp`.
2. Selecciona **Bind flow**.
3. Elige **Browser flow** y pulsa **Save**.
*(A partir de este momento, todo el que intente entrar vía web pasará por tu flujo modificado).*

### 1.4 Prueba de Usuario (Cerrando el ciclo)
1. Abre tu ventana de **Incógnito**.
2. Intenta hacer login con `maria.lopez` en `http://keycloak.corp.local/realms/Corp-Prod/account/`.
3. Verás que Keycloak detiene a María y le muestra un **Código QR**.
4. Usa la app de tu móvil (Google Authenticator o FreeOTP) para escanearlo, introduce el código de 6 dígitos y valida el acceso.

---

## 🔀 Práctica 2: Flujos Condicionales (Fallback)

Obligar a todo el mundo a usar 2FA puede ser problemático para cuentas de servicio o usuarios externos. Vamos a hacerlo inteligente: "Exigir 2FA *SOLO* si el usuario pertenece al grupo Tecnología".

1. Vuelve a **Authentication** -> `browser-corp`.
2. Cambia el **Browser OTP Form** de nuevo a **Conditional**.
3. Pulsa el ícono de "más" (`+`) o "Add execution" al lado de *Browser - Conditional OTP*.
4. Selecciona **Condition - User Role** (o User Group) y pulsa Add.
5. Sube esa condición (con las flechitas) para que quede *justo encima* del paso de OTP.
6. Haz clic en la "tuerca" (Configuración) de tu nueva Condición.
7. Asígnale el alias `Es-Tecnico` y selecciona el Rol o Grupo que creamos en la Sesión 4 (`Tecnologia` o el rol `empleado-base`).
*(Ahora, Keycloak evaluará la condición. Si el usuario la cumple, le pedirá el OTP; si no, le dejará pasar solo con contraseña).*

---

## 🚦 Práctica 3: Autorización Fina (ABAC - Attribute-Based Access Control)

RBAC (Roles) está bien para decir "Eres Contable", pero ABAC nos permite decir: "Eres Contable, *PERO* solo puedes ver este informe si estás en la oficina de Madrid y es horario laboral".

### 3.1 Habilitar Authorization Services en el Cliente
Para usar ABAC, el cliente debe ser "Confidencial" (no público).
1. Ve a **Clients** -> Selecciona `app-contabilidad`.
2. En la pestaña **Settings** -> **Capability config**:
   * Cambia **Client authentication** a `ON`.
   * Cambia **Authorization** a `ON`.
3. Pulsa **Save**. Aparecerá una nueva pestaña llamada **Authorization**.

### 3.2 Crear el Recurso (Resource)
1. Entra en la pestaña **Authorization** -> Subpestaña **Resources**.
2. Pulsa **Create resource**.
3. **Name:** `Reporte Financiero Anual`
4. **URIs:** `/api/reportes/anual`
5. Pulsa **Save**.

### 3.3 Crear la Política (Policy)
Vamos a usar el atributo `centro_coste` que le pusimos a María en la Sesión 4.
1. Ve a la subpestaña **Policies** -> Pulsa **Create policy** -> Selecciona **User**.
2. **Name:** `Solo-Sede-Madrid`
3. Abajo, en la configuración, no seleccionaremos un usuario específico, sino que nos basaríamos en atributos si usáramos JavaScript o Group policies. 
*(Para este ejercicio guiado en la UI, borra esta policy de User, pulsa Create Policy -> **Group** y selecciona el grupo `Ventas`).*
*Nota: Las políticas de atributos (ABAC puras) requieren habilitar las políticas de JavaScript o enviarlas desde el cliente, usaremos Grupos para ver el resultado rápido en la consola.*

### 3.4 Crear el Permiso (Permission) y Evaluar
Un Permiso es el "pegamento" que une el Recurso con la Política.
1. Ve a la subpestaña **Permissions** -> **Create permission** -> **Resource-based**.
2. **Name:** `Permiso-Reporte-Ventas`
3. **Resources:** Selecciona `Reporte Financiero Anual`.
4. **Policies:** Selecciona `Solo-Sede-Madrid`.
5. Pulsa **Save**.

---

## 🛑 Práctica 4: Gestión de Sesiones y Revocación

Como administradores de seguridad, debemos saber cómo reaccionar ante el robo de un portátil o un despido inminente.

1. Ve a la sección **Sessions** en el menú lateral.
2. Aquí verás todas las sesiones activas en todos los clientes.
3. Para expulsar a un usuario específico: Ve a **Users** -> Busca a `maria.lopez` -> Ve a la pestaña **Sessions**.
4. Pulsa **Logout all**. 
*(El token de acceso de María dejará de ser válido inmediatamente y su próxima petición a la aplicación será rechazada, obligándola a loguearse de nuevo).*

---

## 🚀 Reto Extra: WebAuthn (Passkeys / Biometría)

Las contraseñas están muriendo. Keycloak soporta de forma nativa la autenticación con huella dactilar, Windows Hello o FaceID.

1. Ve a **Authentication** -> **Required actions**.
2. Busca **Webauthn Register** y asegúrate de que esté **Enabled**.
3. Ve a un usuario (ej. `maria.lopez`) -> Pestaña **Details** -> Sección *Required user actions*.
4. Selecciona **Webauthn Register** y guarda.
5. Al intentar iniciar sesión, Keycloak pedirá al usuario que registre su dispositivo biométrico. ¡Pruébalo desde un dispositivo con TouchID o Windows Hello!

***
Notas sobre la Sesión 5:

- Flujos de Autenticación: Es vital recordarles siempre que NO editen el flujo original. Si rompen el flujo browser base y se quedan fuera de la consola, tendrán que entrar a la base de datos a arreglarlo. Hacer una copia es la regla de oro de Keycloak.