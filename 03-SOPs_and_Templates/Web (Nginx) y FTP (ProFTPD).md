# 1. Convenciones y Variables

Antes de ejecutar los comandos en esta guía, asegúrese de reemplazar los siguientes marcadores de posición con los valores correspondientes a su entorno:

- `<HOSTNAME>`: Nombre del servidor (ej. `gamma`).

- `<DOMINIO_WEB>`: Nombre de dominio principal (ej. `xeon.cr`).

- `<FQDN>`: Nombre de dominio completamente calificado (ej. `gamma.xeon.cr`).

- `<IP_SERVIDOR>`: Dirección IP estática del servidor (ej. `192.168.106.3`).

- `<MASCARA_RED>`: Máscara de subred (ej. `255.255.255.0`).

- `<PUERTA_ENLACE>`: IP del Gateway de la red (ej. `192.168.106.254`).

- `<IP_DNS>`: Dirección IP del servidor de nombres/DNS (ej. `192.168.106.1`).

- `<INTERFAZ_RED>`: Interfaz de red principal (ej. `enp0s3`).

- `<RUTA_RAIZ_WEB>`: Directorio principal de los archivos del sitio (ej. `/var/www/xeon.cr`).

- `<USUARIO_FTP>`: Usuario encargado de administrar los archivos web vía FTP (ej. `webadmin`).


# 2. Estructura de Paso a Paso: Configuración Base del Sistema

**Acción:** Configurar el nombre del host. **Comando:**
```
hostnamectl set-hostname <HOSTNAME>
```

**Acción:** Mapear el nombre del host localmente. **Comando:**
```
nano /etc/hosts
	127.0.0.1       localhost
	127.0.1.1       <FQDN>   <HOSTNAME>
```

**Acción:** Configurar la IP estática. **Comando:**
```
nano /etc/network/interfaces
	# Interfaz de red principal
	allow-hotplug <INTERFAZ_RED>
	iface <INTERFAZ_RED> inet static
	    address <IP_SERVIDOR>
	    netmask <MASCARA_RED>
	    gateway <PUERTA_ENLACE>
```

**Acción:** Definir la resolución DNS. **Comando:*
```
nano /etc/resolv.conf
	nameserver <IP_DNS>
	search <DOMINIO_WEB>
```

**Acción:** Actualizar repositorios e instalar paquetes base. **Comando:**
```
apt update && apt upgrade -y
```

## 3. Despliegue y Configuración de Nginx

**Acción:** Instalar Nginx y crear el directorio raíz para el sitio web. **Comandos:**
```
apt install nginx -y
mkdir -p <RUTA_RAIZ_WEB>/html
```

**Acción:** Transferir y asegurar los certificados SSL/TLS emitidos por su CA.
**Comandos:**
Asumiendo que los archivos ya fueron transferidos al servidor
```
mv <LLAVE_SSL>.key /etc/ssl/private/
chown root:root /etc/ssl/private/<LLAVE_SSL>.key
chmod 600 /etc/ssl/private/<LLAVE_SSL>.key

mv <CERTIFICADO_CA>.crt /etc/ssl/certs/
chown root:root /etc/ssl/certs/<CERTIFICADO_CA>.crt
chmod 644 /etc/ssl/certs/<CERTIFICADO_CA>.crt

mv <CERTIFICADO_SSL>.crt /etc/ssl/certs/
chown root:root /etc/ssl/certs/<CERTIFICADO_SSL>.crt
chmod 644 /etc/ssl/certs/<CERTIFICADO_SSL>.crt
```

**Acción:** Crear el archivo de configuración del sitio web (Virtual Host) forzando HTTPS.
**Comando:**
```
nano /etc/nginx/sites-available/<DOMINIO_WEB>
```
[[<DOMINIO_WEB>]]

**Acción:** Habilitar el sitio mediante un enlace simbólico. 
**Comando:**
```
ln -s /etc/nginx/sites-available/<DOMINIO_WEB> /etc/nginx/sites-enabled/
```

# 4. Gestión de Usuarios y Permisos para FTP/Web

**Acción:** Crear un usuario de sistema sin acceso a shell interactiva, pero con permisos para administrar el directorio web. 
**Comandos:**
```
useradd -m -d <RUTA_RAIZ_WEB> -s /usr/sbin/nologin <USUARIO_FTP>
passwd <USUARIO_FTP>
chown -R <USUARIO_FTP>:www-data <RUTA_RAIZ_WEB>
chmod -R 775 <RUTA_RAIZ_WEB>
```

# 5. Despliegue y Configuración de ProFTPD

**Acción:** Instalar el servidor ProFTPD y su módulo criptográfico para soporte TLS.
**Comando:**
```
apt install proftpd-basic proftpd-mod-crypto -y
```

**Acción:** Configurar los parámetros principales del servidor FTP, enjaulando al usuario y restringiendo accesos.
**Comando:**
```
nano /etc/proftpd/proftpd.conf
```
[[proftpd.conf]]

**Acción:** Configurar el módulo TLS de ProFTPD para habilitar FTPS explícito y forzar conexiones cifradas.
**Comando:**
```
nano /etc/proftpd/tls.conf
```
[[tls.conf]]

**Acción:** Habilitar configuraciones necesarias de módulos.
**Comando:**
```
nano /etc/proftpd/modules.conf
```
_(Nota: Verifique que la línea `LoadModule mod_tls.c` esté sin comentar)._

## 6. Sección de Gestión, Verificación y Troubleshooting
Para asegurar que ambos servicios funcionen correctamente después de aplicar las configuraciones, utilice los siguientes comandos:

**Verificación de Sintaxis (Obligatorio antes de reiniciar):**

- **Nginx:** `nginx -t`

- **ProFTPD:** `proftpd -t`

**Comandos de Control de Servicios:**

- **Nginx:** `systemctl start | stop | restart | reload | status nginx`

- **ProFTPD:** `systemctl start | stop | restart | reload | status proftpd`


**Troubleshooting y Logs:**

- **Revisión en tiempo real (Systemd):**
    
    - `journalctl -u nginx`
    
    - `journalctl -u proftpd`
    
- **Rutas de Logs:**
    
    - Errores web: `tail -f /var/log/nginx/error.log` (o el log específico definido como `/var/log/nginx/<DOMINIO_WEB>.error.log`).
    
    - Eventos FTP: `tail -f /var/log/proftpd/proftpd.log`.
    
    - Eventos TLS FTP: `tail -f /var/log/proftpd/tls.log`.
    
- **Errores Comunes:**
    
    - **Nginx - Error 403 Forbidden:** Suele ocurrir si los permisos del directorio `<RUTA_RAIZ_WEB>` no son correctos o falta el archivo `index.html`. Asegúrese de que el usuario `www-data` pueda leer los archivos.
    
    - **ProFTPD - Fallo en modo pasivo:** Ocurre comúnmente detrás de NAT. Asegúrese de que el rango de `PassivePorts` (ej. 49152-65534) esté abierto en el firewall (UFW/iptables).
    
    - **ProFTPD - Fallo de conexión TLS:** Asegúrese de que el cliente FTP (como FileZilla) esté configurado para requerir "FTP explícito sobre TLS".
    

## 7. Funciones Avanzadas y Próximos Pasos

Con la configuración aplicada, el entorno ya cuenta con una base sólida de seguridad perimetral, habiendo implementado exitosamente el cifrado HTTPS forzado mediante Nginx y el soporte para sesiones seguras obligatorias en ProFTPD. Para continuar evolucionando esta arquitectura hacia estándares de alta disponibilidad y producción, considere:

- **Seguridad Activa (Fail2Ban):** Implementar políticas de Fail2Ban para monitorear activamente los logs de acceso de Nginx y ProFTPD, bloqueando automáticamente a nivel de firewall (iptables/UFW) las direcciones IP que realicen múltiples intentos fallidos de autenticación o muestren patrones maliciosos.

- **Gestión y Rotación de Logs (Logrotate):** Establecer reglas estrictas de rotación, compresión y retención para los directorios `/var/log/nginx/` y `/var/log/proftpd/`, garantizando que el almacenamiento del servidor no se sature a lo largo del tiempo.

- **Automatización de Certificados SSL/TLS:** Desarrollar un flujo de trabajo (mediante scripts cron o Ansible) para la renovación y despliegue automático de los certificados de la CA interna antes de su expiración, reduciendo el margen de error humano.

- **Alta Disponibilidad y Balanceo de Carga:** Utilizar el servidor Nginx actual como proxy inverso (Reverse Proxy) y balanceador de carga (Load Balancer) para distribuir el tráfico web entrante hacia un clúster de servidores backend.

- **Telemetría y Monitoreo:** Integrar agentes de monitoreo de métricas (como Prometheus con Node Exporter, o Netdata) para visualizar en tiempo real el consumo de CPU/RAM, ancho de banda de red, y la cantidad de conexiones simultáneas tanto web como FTP.