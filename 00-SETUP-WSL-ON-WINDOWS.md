# Instalación de WSL en Windows y solución de problemas con virtualización e hipervisor

Este tutorial explica paso a paso cómo instalar **WSL (Windows Subsystem for Linux)** en **Windows 10 y Windows 11**, verificar que funciona correctamente con **WSL 2**, y resolver los problemas más comunes relacionados con **virtualización, BIOS/UEFI y el hipervisor**.

---

# 1. Requisitos previos

Antes de instalar WSL, asegúrate de cumplir los siguientes requisitos.

### Versión de Windows compatible

- **Windows 11** (cualquier edición de escritorio)
- **Windows 10 versión 2004 o superior** (compilación **19041** o superior)

Puedes comprobar tu versión ejecutando:

```powershell
winver
```

---

### Permisos de administrador

Necesitarás ejecutar algunos comandos como **Administrador**:

* **PowerShell**
* **Windows Terminal**
* **Símbolo del sistema (cmd)**

Para abrir PowerShell como administrador:

1. Abre el menú **Inicio**.
2. Escribe `PowerShell`.
3. Haz clic derecho sobre **Windows PowerShell**.
4. Selecciona **Ejecutar como administrador**.

---

### Conexión a Internet

Es necesaria para:

* Descargar el **kernel de WSL**
* Instalar una **distribución de Linux**

---

# 2. Verificar que la virtualización está habilitada

WSL 2 requiere **virtualización por hardware**.

### Verificar desde el Administrador de tareas

1. Presiona:

```
Ctrl + Shift + Esc
```

2. Ve a:

```
Rendimiento → CPU
```

3. Busca la línea:

```
Virtualización
```

Debe mostrar:

```
Habilitado
```

---

### Verificar desde línea de comandos

También puedes usar:

```powershell
systeminfo | find "Virtualization"
```

Si aparece que la virtualización está deshabilitada, deberás activarla en la **BIOS/UEFI** (ver sección posterior).

---

# 3. Instalar WSL

En versiones modernas de Windows, WSL puede instalarse con un solo comando.

Abre **PowerShell como administrador** y ejecuta:

```powershell
wsl --install
```

Este comando:

* Activa **Windows Subsystem for Linux**
* Activa **Virtual Machine Platform**
* Instala el **kernel de WSL**
* Instala **Ubuntu** por defecto

---

### Instalar una distribución específica

Si prefieres instalar una versión concreta de Ubuntu:

```powershell
wsl --install -d Ubuntu-22.04
```

---

### Ver distribuciones disponibles

```powershell
wsl --list --online
```

Ejemplo de salida:

```
Ubuntu
Ubuntu-22.04
Debian
kali-linux
openSUSE
```

---

### Instalar WSL sin distribución

Si quieres instalar primero el entorno base:

```powershell
wsl --install --no-distribution
```

Luego podrás instalar distribuciones desde Microsoft Store o mediante `wsl`.

---

# 4. Reiniciar el sistema

Después de ejecutar `wsl --install`, **reinicia el ordenador**.

Tras el reinicio:

* Se abrirá automáticamente la configuración de la distribución.
* Se pedirá crear:

```
Usuario Linux
Contraseña
```

Ejemplo:

```
Enter new UNIX username:
Enter new UNIX password:
```

---

# 5. Actualizar WSL

Después de la instalación, es recomendable actualizar los componentes de WSL.

Ejecuta:

```powershell
wsl --update
```

Para comprobar el estado:

```powershell
wsl --status
```

---

# 6. Verificar que la distribución usa WSL 2

Para listar las distribuciones instaladas:

```powershell
wsl -l -v
```

Ejemplo:

```
NAME            STATE           VERSION
Ubuntu-22.04    Running         2
```

La columna **VERSION** debe mostrar:

```
2
```

---

### Convertir una distribución a WSL 2

Si aparece versión **1**, puedes actualizarla:

```powershell
wsl --set-version Ubuntu-22.04 2
```

---

### Establecer WSL 2 como versión por defecto

```powershell
wsl --set-default-version 2
```

---

# 7. Problema común: virtualización o hipervisor deshabilitado

Uno de los errores más comunes es:

```
WSL 2 requires an update to its kernel component
```

o

```
Virtualization is disabled in the firmware
```

o

```
WslRegisterDistribution failed with error: 0x80370102
```

Esto suele ocurrir cuando:

* La **virtualización está deshabilitada en la BIOS**
* El **hipervisor de Windows no está habilitado**

---

# 8. Activar el hipervisor desde Windows

Abre **Símbolo del sistema como administrador** y ejecuta:

```cmd
bcdedit /set hypervisorlaunchtype auto
```

Si el comando se ejecuta correctamente, verás:

```
The operation completed successfully.
```

Después **reinicia el ordenador**.

---

# 9. Activar virtualización en la BIOS/UEFI

Si la virtualización sigue deshabilitada, deberás activarla en la BIOS.

### Pasos generales

1. Reinicia el ordenador.
2. Antes de que arranque Windows presiona repetidamente:

```
F2
F10
F12
Esc
Delete
```

(depende del fabricante)

---

### Buscar opciones de virtualización

En la BIOS busca secciones como:

```
Advanced
CPU Configuration
Processor
System Configuration
```

Las opciones suelen llamarse:

**Intel**

```
Intel Virtualization Technology
Intel VT-x
```

**AMD**

```
SVM Mode
AMD-V
```

Cambia el valor a:

```
Enabled
```

---

### Guardar cambios

Normalmente se hace con:

```
F10 → Save and Exit
```

El sistema reiniciará automáticamente.

---

# 10. Activar características manualmente (si `wsl --install` falla)

En algunos sistemas antiguos puede ser necesario habilitar manualmente los componentes.

Ejecuta en **PowerShell administrador**:

```powershell
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

y

```powershell
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Después reinicia el sistema.

---

# 11. Hyper-V (opcional)

**WSL 2 no requiere Hyper-V completo**, pero algunos entornos lo utilizan, por ejemplo:

* Docker Desktop (modo Hyper-V)
* herramientas de virtualización avanzadas

Para habilitarlo:

```powershell
dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart
```

Después reinicia el sistema.

---

# 12. Verificación final

Comprueba que WSL funciona correctamente.

### Iniciar WSL

```powershell
wsl
```

Deberías ver algo como:

```
usuario@PC:~$
```

---

### Verificar distribuciones

```powershell
wsl -l -v
```

---

### Ver estado de WSL

```powershell
wsl --status
```

---

# 13. Errores comunes y soluciones

### Error

```
WslRegisterDistribution failed with error: 0x80370102
```

Causa:

* Virtualización deshabilitada en BIOS.

Solución:

* Activar virtualización.

---

### Error

```
The Windows Subsystem for Linux has not been enabled
```

Solución:

```powershell
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux
```

---

### Error

```
Kernel update required
```

Solución:

```powershell
wsl --update
```

---

### Error

```
wsl --install requires Windows 10 version 2004 or higher
```

Solución:

Actualizar Windows.

---

# 14. Resumen rápido

Pasos principales:

1. Verificar que **Windows es compatible** (`winver`)
2. Verificar que **virtualización está habilitada**
3. Ejecutar:

```powershell
wsl --install
```

4. Reiniciar el sistema
5. Crear usuario Linux
6. Actualizar WSL:

```powershell
wsl --update
```

7. Verificar versión:

```powershell
wsl -l -v
```

Si todo está correcto, WSL estará listo para usar.

Podrás instalar herramientas como:

* Node.js
* Docker CLI
* Git
* Python
* entornos de desarrollo completos de Linux dentro de Windows.
