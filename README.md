# 🛡️ Keycloak: Seguridad Centralizada, Gestión de Identidades y Control de Acceso

¡Bienvenidos al repositorio oficial del curso! 

Este espacio contiene todo el material, manuales de preparación, manifiestos de infraestructura y código fuente que utilizaremos a lo largo de las 30 horas de formación. El curso está diseñado para cubrir el ciclo de vida completo de una arquitectura IAM empresarial (Identity and Access Management) basada en **Keycloak**.

---

## 🎯 Objetivo del Curso

Aprender a desplegar, integrar y administrar Keycloak como solución IAM empresarial, con foco en **seguridad, personalización (Keycloakify) y alta disponibilidad (HA)** en entornos Kubernetes On-Premise (RKE2).

**Stack Tecnológico:**

- 🔐 **IAM:** Keycloak
- ☸️ **Infraestructura:** Docker, Kubernetes (RKE2 / Rancher), NGINX Ingress
- ⚙️ **Automatización:** Ansible
- ☕ **Backend:** Java 25 LTS (Quarkus)
- ⚛️ **Frontend:** Node.js 24 LTS (React)
- 🗄️ **Persistencia:** PostgreSQL LTS

---

## 🚀 PASO 1: Preparación del Entorno (Start Here)

Dado que en este curso participan distintos perfiles profesionales, hemos dividido las instrucciones de preparación. **Es estrictamente obligatorio completar el manual correspondiente a tu perfil antes de la Sesión 1.**

> 🪟 **¿Usas Windows?**
> Antes de abrir el manual de tu perfil, debes configurar el Subsistema de Windows para Linux. Sigue esta guía primero:
> 👉 **[00 - Guía de Instalación de WSL2 en Windows](./00-SETUP-WSL-ON-WINDOWS.md)**

### Elige tu Perfil Profesional:


| Perfil                      | Descripción                                                                                            | Guía de Instalación                                         |
| --------------------------- | ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| 🏗️ **Infraestructura**     | DevOps, SysAdmins, Arquitectos Cloud. Encargados del despliegue en RKE2, Ansible, HA y Monitorización. | [👉 **Ver Manual de Infraestructura](./01-SETUP-INFRA.md)** |
| 💻 **Desarrolladores**      | Backend y Frontend. Encargados de integrar Quarkus, React, OIDC y customizar temas con Keycloakify.    | [👉 **Ver Manual de Desarrollo](./02-SETUP-DEV.md)**        |
| 👥 **Usuarios / Seguridad** | Administradores IAM, RRHH, Ciberseguridad. Encargados de políticas ABAC/RBAC, MFA y auditoría.         | [👉 **Ver Manual de Usuarios](./03-SETUP-USUARIO.md)**      |


---

## 📅 Roadmap del Curso (8 Sesiones)

El temario está estructurado en un flujo evolutivo donde cada equipo construye sobre el trabajo del anterior. Cada sesión dispone de un **manual de prácticas** con los pasos detallados; en la tabla siguiente se resumen los temas y el enlace al documento de cada sesión.

| Sesión | Bloque | Contenido / Temas | Manual de prácticas |
|--------|--------|-------------------|---------------------|
| **S1** | 🏗️ Infraestructura | Fundamentos IAM, Docker base y creación de Imagen Estandarizada (Quarkus). | [S1 - Prácticas Infra](./Sesion-01/S1-PRACTICAS-INFRA.md) |
| **S2** | 🏗️ Infraestructura | Despliegue en Kubernetes On-Premise (RKE2), NGINX Ingress y Monitorización (Grafana/Prometheus). | [S2 - Prácticas Infra](./Sesion-02/S2-PRACTICAS-INFRA.md) |
| **S3** | 🏗️ Infraestructura | Gestión Multi-Tenant, integraciones LDAP, Backups y estrategias de actualización Zero-Downtime. | [S3 - Prácticas Infra](./Sesion-03/S3-PRACTICAS-INFRA.md) |
| **S4** | 👥 Usuarios y gestión | Fundamentos de la Consola Admin, diseño de modelo RBAC, Grupos, Roles Compuestos y Mapeos. | [S4 - Prácticas Usuarios](./Sesion-04/S4-PRACTICAS-USUARIOS.md) |
| **S5** | 👥 Usuarios y gestión | Flujos de Autenticación, MFA (Doble Factor) obligatorio/condicional y Políticas de Control de Acceso (ABAC). | [S5 - Prácticas Usuarios](./Sesion-05/S5-PRACTICAS-USUARIOS.md) |
| **S6** | 💻 Desarrollo | Integración Backend: Java Quarkus LTS, validación de Tokens OIDC y protección de APIs REST. | [S6 - Prácticas Devs Backend](./Sesion-06/S6-PRACTICAS-DEVS-BACKEND.md) |
| **S7** | 💻 Desarrollo | Integración Frontend: React LTS, flujos PKCE, Identity Brokering y branding corporativo con **Keycloakify**. | [S7 - Prácticas Devs Frontend](./Sesion-07/S7-PRACTICAS-DEVS-FRONTEND.md) |
| **S8** | 🚀 Consolidación | Proyecto Final Evolutivo: Implementación E2E de un caso de uso realista ("NeoBank") uniendo RKE2, políticas estrictas y aplicaciones integradas. | [S8 - Proyecto Final](./Sesion-08/S8-PROYECTO-FINAL.md) |

---

*Instructor: Robson Paradella Rocha - Imagina Formacion.*  
*Formación impartida para la adopción de tecnologías Open Source Enterprise.*