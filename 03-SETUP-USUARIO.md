# 👥 Manual de Preparación de Entorno: Equipo de Usuarios y Seguridad

Bienvenido al curso de **Keycloak: Seguridad Centralizada y Gestión de Identidades**. Este documento detalla los preparativos necesarios para tu perfil. Durante el curso, tu rol será gestionar la consola de administración, definir políticas de seguridad (RBAC/ABAC), configurar el Doble Factor de Autenticación (MFA) y auditar los accesos.

A diferencia de los perfiles de Infraestructura y Desarrollo, no necesitas instalar motores de contenedores ni lenguajes de programación. Tu entorno de trabajo será el navegador web y tu dispositivo móvil.

## 💻 1. Requisitos de Hardware y Sistema Operativo

* **Equipo:** Cualquier PC o Mac moderno con Windows, macOS o Linux.
* **Conexión:** Acceso a internet estable.
* **Permisos:** No requieres permisos de administrador local en tu máquina para estas herramientas.

---

## 🌐 2. Navegador Web Moderno

Todo el trabajo de configuración de Realms, Usuarios, Grupos y Políticas se realiza a través de la interfaz gráfica de Keycloak (Admin Console).

* **Requisito:** Tener instalado y actualizado un navegador web moderno basado en Chromium o Firefox.
  * Google Chrome, Microsoft Edge, Brave o Mozilla Firefox.
* **Recomendación:** Ten a mano la funcionalidad de **"Ventana de Incógnito" o "Navegación Privada"**. La usaremos constantemente para probar los inicios de sesión simulando ser un usuario final sin cerrar nuestra sesión de administrador.

---

##📱 3. Aplicación de Doble Factor (MFA / OTP)

En la Sesión 5 configuraremos y forzaremos el uso del Doble Factor de Autenticación (2FA) para proteger cuentas críticas. Necesitarás una aplicación en tu teléfono móvil capaz de escanear códigos QR y generar tokens temporales (TOTP).

* **Requisito:** Instala una de las siguientes aplicaciones en tu smartphone (iOS o Android):
  * **Google Authenticator** (Recomendada)
  * **Microsoft Authenticator**
  * **FreeOTP** (De código abierto, respaldada por Red Hat)
  * **Authy**

![Google Authenticator](images/google_authenticator.png)

---

## 🛠️ 4. Herramientas de Diagnóstico (En la Nube)

Para entender cómo funciona la autorización moderna, auditaremos los "Tokens" de seguridad que emite Keycloak para verificar qué permisos se están otorgando realmente a los usuarios.

* **Guarda en tus marcadores (Favoritos):** [https://jwt.io/](https://jwt.io/)
* Usaremos esta web gratuita desarrollada por Auth0 para pegar nuestros tokens y decodificar su contenido (el *Payload* y los *Claims*) de forma visual y sencilla.

---

## 📹 5. Comunicaciones (Zoom)

Como en todos los perfiles del curso, las sesiones requieren el cliente de escritorio de **Zoom** instalado y actualizado.

* **Vital para las prácticas:** Se recomienda el uso de **dos pantallas** (una para ver la pantalla del instructor y otra para tu propia consola de administración y navegador).
* Asegúrate de tener cámara, micrófono y altavoces probados antes de iniciar la primera sesión.