Documentación técnica, cmdlets y procedimientos para la administración y configuración de entornos Windows.

## Instalación y Configuración Inicial

### Omitir cuenta de Microsoft (Cuenta Local)

Comando utilizado durante la experiencia de configuración inicial (OOBE) para forzar la creación de una cuenta local sin requerir conexión a internet o inicio de sesión de Microsoft.

- **Ejecución:** Presionar `Shift + F10` en la pantalla de configuración para abrir CMD y ejecutar:

```
start ms-cxh:localonly
```
_(Nota: Alternativamente, en versiones recientes de Windows 11 se suele utilizar `OOBE\BYPASSNRO` para deshabilitar el requisito de red)._

## Sysprep (System Preparation Tool)

**Casos de Uso:**

-  Creación de imágenes de Windows (Golden Images) para despliegue masivo.

-  Restablecer el SID (Security Identifier) del equipo.


**Pasos para realizar la generalización:**

1. Desactivar cifrado de disco (BitLocker) si está habilitado: `Disable-BitLocker -MountPoint "C:"`

2. Verificar que no haya paquetes Appx de usuario bloqueando el proceso (revisar `setupact.log`).

3. Ejecutar el comando de preparación desde una terminal con privilegios de Administrador:


```PowerShell
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

## PowerShell: Administración y Cmdlets

### Gestión de Usuarios y Grupos Locales
```PowerShell
# Listar todos los usuarios locales
Get-LocalUser

# Crear un nuevo usuario local con contraseña
$Password = Read-Host -AsSecureString
New-LocalUser "NombreUsuario" -Password $Password -FullName "Nombre Completo" -Description "Descripción"

# Habilitar o deshabilitar un usuario
Enable-LocalUser -Name "NombreUsuario"
Disable-LocalUser -Name "NombreUsuario"

# Añadir un usuario al grupo de Administradores
Add-LocalGroupMember -Group "Administrators" -Member "NombreUsuario"
```

### Gestión de Directorios y Archivos
```PowerShell
# Listar contenido de un directorio (equivalente a 'ls' o 'dir')
Get-ChildItem -Path C:\Ruta\Al\Directorio

# Crear una nueva carpeta
New-Item -Path "C:\Ruta" -Name "NuevaCarpeta" -ItemType "directory"

# Crear un archivo vacío
New-Item -Path "C:\Ruta" -Name "archivo.txt" -ItemType "file"

# Copiar archivos de forma recursiva
Copy-Item -Path C:\Origen\* -Destination C:\Destino -Recurse

# Eliminar un archivo o carpeta (forzar eliminación de ocultos/solo lectura)
Remove-Item -Path C:\Ruta\Archivo.txt -Force
```

### Permisos (Listas de Control de Acceso - ACL)
```PowerShell
# Ver los permisos actuales de un archivo o carpeta
Get-Acl -Path C:\Ruta\Archivo.txt | Format-List

# Ejemplo para copiar permisos de una carpeta a otra
$Permisos = Get-Acl -Path C:\CarpetaOrigen
Set-Acl -Path C:\CarpetaDestino -AclObject $Permisos
```
_(Nota: Para operaciones complejas de permisos, la herramienta por línea de comandos `icacls` a menudo se integra dentro de los scripts de PowerShell)._

### Información del Sistema
```PowerShell
# Obtener información general del hardware y SO
Get-ComputerInfo

# Ver el estado de la red (IP, MAC, Gateway)
Get-NetIPAddress
Get-NetAdapter

# Listar los procesos que consumen más CPU (Top 5)
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5

# Ver el espacio disponible en los discos lógicos
Get-Volume
```

### Administración General, Servicios y Redes
```PowerShell
# Listar servicios en ejecución
Get-Service | Where-Object Status -eq "Running"

# Reiniciar un servicio específico (ej. DNS Client)
Restart-Service -Name dnscache

# Probar conectividad y puertos (equivalente avanzado a ping/telnet)
Test-NetConnection -ComputerName google.com -Port 443

# Consultar el visor de eventos (ej. Errores de sistema recientes)
Get-WinEvent -LogName System -MaxEvents 20 | Where-Object LevelDisplayName -eq "Error"
```
