```
# Carga de módulos dinámicos
Include /etc/proftpd/modules.conf

# Soporte IPv6 activado
UseIPv6 on

<IfModule mod_ident.c>
    # Desactivar peticiones de identificación para evitar demoras en la conexión
    IdentLookups off
</IfModule>

# Nombre del servidor que verán los clientes al conectarse
ServerName "<HOSTNAME> FTP Server"

# Configuración del demonio como servidor independiente principal
ServerType standalone
DefaultServer on

# Tiempos máximos de espera (en segundos)
TimeoutNoTransfer 600
TimeoutStalled 600
TimeoutIdle 1200

# Opciones de presentación y filtros de seguridad
DeferWelcome off
ShowSymlinks on
DisplayLogin welcome.msg
DisplayChdir .message true
ListOptions "-l"
DenyFilter \*.*/

# Enjaular a todos los usuarios dentro de sus directorios base por seguridad (Chroot)
DefaultRoot ~

# Permitir el acceso a usuarios que no poseen una shell interactiva (ej. /usr/sbin/nologin)
RequireValidShell off

# Puerto estándar de conexión FTP
Port 21

# Rango de puertos pasivos para conexiones a través de Firewalls o NAT (Descomentar si aplica)
# PassivePorts <PUERTOS_PASIVOS_INICIO> <PUERTOS_PASIVOS_FIN>

# Límite de conexiones simultáneas para mitigar ataques DoS
MaxInstances 30

# Usuario y grupo bajo los que opera el servicio ProFTPD
User proftpd
Group nogroup

# Máscara de permisos por defecto para los archivos/directorios nuevos y sobreescritura
Umask 022 022
AllowOverwrite on

# Rutas de los logs de transferencias y eventos generales del servicio
TransferLog /var/log/proftpd/xferlog
SystemLog /var/log/proftpd/proftpd.log

<IfModule mod_delay.c>
    # Mitigación de ataques de temporización (Timing Attacks)
    DelayEngine on
</IfModule>

# Inclusión del archivo de configuración para conexiones seguras (FTPS)
Include /etc/proftpd/tls.conf

# Bloque de seguridad: Restringir el inicio de sesión FTP ÚNICAMENTE a nuestro usuario web
<Limit LOGIN>
  AllowUser <USUARIO_FTP>
  DenyAll
</Limit>

# Directorio para otras configuraciones personalizadas
Include /etc/proftpd/conf.d/
```
