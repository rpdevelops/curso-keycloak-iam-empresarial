# 👥 Prácticas Sesión 4: Fundamentos y Gestión RBAC Orientada a Equipos

En esta sesión dejaremos atrás la consola de comandos (terminal) y trabajaremos 100% desde la **Consola de Administración de Keycloak**. Nuestro objetivo es traducir el organigrama de una empresa real y sus políticas de seguridad a la arquitectura de Keycloak, gestionando usuarios, grupos, roles y contraseñas.

---

## 🏢 Práctica 1: Estructura Organizativa (Grupos y Usuarios)

Vamos a crear los departamentos base de nuestra empresa y a los primeros empleados. 

### 1.1 Creación de Grupos

Los grupos en Keycloak sirven para reflejar la estructura de la empresa y facilitar la asignación masiva de permisos.

1. Accede a tu Consola de Administración (ej. `http://keycloak.corp.local`) y selecciona el Realm **Corp-Prod** (creado en la sesión anterior).
2. Ve a la sección **Groups** en el menú lateral.
3. Haz clic en **Create group**.
4. Crea tres grupos base: `Ventas`, `RRHH` y `Tecnologia`.
5. *Subgrupos:* Haz clic en el grupo `Tecnologia`, selecciona el botón de opciones (tres puntos) o "Create child group" y crea el subgrupo `Soporte-N1`.

### 1.2 Creación de Usuarios y Políticas de Contraseñas

1. Ve a la sección **Users** y pulsa **Add user**.
2. Crea al usuario `maria.lopez` (Email: `maria@corp.local`, First Name: `Maria`, Last Name: `Lopez`).
3. Asegúrate de que **Email verified** esté en `Yes` (para evitar envíos de correos por ahora) y pulsa **Create**.
4. Ve a la pestaña **Credentials** de Maria y pulsa **Set password**.
5. Escribe una contraseña temporal (ej. `temporal123`).
6. **Importante:** Deja activada la opción **Temporary**. Esto obligará a María a cambiar su contraseña la primera vez que inicie sesión, cumpliendo con las políticas de seguridad.
7. Ve a la pestaña **Groups**, pulsa **Join Group** y añade a María al grupo `Ventas`.

---

## 🔐 Práctica 2: Diseño del Modelo RBAC (Roles y Jerarquías)

El Control de Acceso Basado en Roles (RBAC) dicta qué puede hacer cada persona. En Keycloak existen **Realm Roles** (Permisos globales para toda la empresa) y **Client Roles** (Permisos exclusivos de una aplicación concreta).

### 2.1 Roles de Aplicación (Client Roles)

Vamos a simular que tenemos una aplicación de contabilidad.

1. Ve a **Clients** y pulsa **Create client**.
2. **Client ID:** `app-contabilidad`. Pulsa **Save**.
3. Dentro de `app-contabilidad`, ve a la pestaña **Roles** y pulsa **Create role**.
4. Crea dos roles: `ver-facturas` y `editar-facturas`.

### 2.2 Roles Globales y Roles Compuestos (Jerarquía)

Un rol compuesto actúa como una "matrioska": contiene a otros roles dentro de sí mismo.

1. Ve a **Realm Roles** en el menú lateral principal.
2. Crea el rol `empleado-base`.
3. Crea otro rol llamado `director-financiero`.
4. Entra en el rol `director-financiero`, ve al menú de acciones superior (o pestaña *Associated roles*) y pulsa **Add associated roles**.
5. Marca el filtro para buscar por clientes (*Filter by clients*), busca `app-contabilidad` y selecciona los roles `ver-facturas` y `editar-facturas`.
6. Añade también el rol global `empleado-base`.

*(Ahora, cualquiera que sea 'director-financiero' heredará automáticamente todos esos permisos técnicos).*

### 2.3 Asignación de Roles a Grupos (Mejor Práctica)

Nunca asignes permisos uno a uno a los usuarios. Asígnalos a los Grupos.

1. Ve a **Groups** -> Selecciona el grupo `Ventas`.
2. Ve a la pestaña **Role mapping** y pulsa **Assign role**.
3. Asígnale el rol global `empleado-base`.
4. **Comprobación:** Ve a **Users** -> Busca a `maria.lopez` -> Ve a su pestaña **Role mapping**. Desmarca "Hide inherited roles" (o mira sus "Effective roles"). Verás que María tiene el rol `empleado-base` heredado directamente de su grupo.

---

## 🏷️ Práctica 3: Atributos Personalizados y Protocol Mappers

A veces los roles no son suficientes y las aplicaciones necesitan datos extra del usuario (ej. su Centro de Coste o Departamento) para tomar decisiones de negocio. Vamos a crear un dato y usar un **Mapper** para inyectarlo en el Token de seguridad.

### 3.1 Añadir el Atributo al Usuario
1. Ve a **Users** y selecciona a `maria.lopez`.
2. Ve a la pestaña **Attributes**.
3. Añade la siguiente clave-valor:
   * **Key:** `centro_coste`
   * **Value:** `VENTAS-MADRID-01`
4. Pulsa **Save**.

### 3.2 Crear el Mapper (Inyección en el Token)
Ahora le diremos a Keycloak que cada vez que alguien haga login en la app de contabilidad, busque ese atributo y lo meta en el "carnet de identidad" (JWT) del usuario.
1. Ve a **Clients** en el menú lateral y selecciona `app-contabilidad`.
2. Ve a la pestaña **Client scopes**.
3. Haz clic en el scope dedicado de la app (se llamará `app-contabilidad-dedicated`).
4. Pulsa el botón **Add mapper** y selecciona **By configuration**.
5. En la lista, busca y selecciona **User Attribute**.
6. Rellena el formulario con estos datos exactos:
   * **Name:** `Mapper-Centro-Coste`
   * **User Attribute:** `centro_coste` *(Debe coincidir exactamente con la Key que le pusimos a María).*
   * **Token Claim Name:** `empresa_centro_coste` *(Así es como se llamará la variable dentro del JSON que leerán los programadores).*
   * **Claim JSON Type:** `String`
   * Asegúrate de que los interruptores **Add to ID token** y **Add to access token** estén en `ON`.
7. Pulsa **Save**.

### 3.3 Validación del Mapper (Evaluate)
¡No hace falta programar una app para ver si funciona! Keycloak tiene una herramienta de simulación.
1. Vuelve a la pestaña **Client scopes** dentro de `app-contabilidad`.
2. Pulsa el botón superior **Evaluate**.
3. En **Users**, selecciona a `maria.lopez`.
4. Pulsa **Evaluate**.
5. Ve a la pestaña **Generated access token**. 
6. Si miras el JSON generado, verás mágicamente la línea: `"empresa_centro_coste": "VENTAS-MADRID-01"`. ¡El equipo de desarrollo ya tiene el dato que necesitaba!

---

## 🕵️ Reto Extra: Auditoría y Eventos de Seguridad

Si un usuario pierde su acceso o hay un intento de ataque, el equipo de Seguridad necesita revisar los logs. Por defecto, Keycloak no guarda eventos de usuario a largo plazo para ahorrar base de datos. ¡Vamos a activarlo!

### Paso 1: Habilitar Eventos

1. Ve a **Realm settings** en el menú lateral.
2. Ve a la pestaña **Events**.
3. En la sección *User events settings*, cambia **Save events** a `ON`.
4. Cambia la expiración a `30 Days` y pulsa **Save**.
5. Haz lo mismo en la sección *Admin events settings* (pon **Save events** a `ON` e incluye *Include representation*).

### Paso 2: Forzar un evento de error

1. Abre una ventana de **Incógnito** en tu navegador.
2. Accede a la URL de cuenta del usuario (Ej: `http://keycloak.corp.local/realms/Corp-Prod/account/`).
3. Intenta iniciar sesión como `maria.lopez` pero usando una contraseña incorrecta tres veces seguidas.

### Paso 3: Revisar la Auditoría

1. Vuelve a tu ventana principal de administrador.
2. Ve a **Events** (en el menú lateral, bajo la sección Manage).
3. Podrás ver en rojo los eventos `LOGIN_ERROR`.
4. Si haces clic en uno de ellos, podrás ver la IP del atacante, el navegador utilizado y el motivo exacto del fallo (`invalid_user_credentials`).

***

### Notas:
- Es vital asignar roles a grupos, no a usuarios, pues es el error número 1 en las empresas que adoptan Keycloak y causa un mantenimiento imposible a largo plazo.
