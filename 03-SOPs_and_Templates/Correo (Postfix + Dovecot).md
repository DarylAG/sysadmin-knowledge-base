# 1. Convenciones y Variables

Antes de iniciar el despliegue, asegúrate de reemplazar los siguientes marcadores de posición en los comandos y archivos de configuración con los valores reales de tu entorno:

- `<DOMINIO_PRINCIPAL>`: El nombre de dominio principal para el correo (ej. `dominio.com`).

- `<HOSTNAME_CORREO>`: El nombre corto del servidor (ej. `mail` o `beta`).

- `<FQDN_CORREO>`: El nombre de dominio completamente calificado del servidor (ej. `mail.dominio.com`).

- `<IP_SERVIDOR>`: Dirección IP estática asignada a este servidor (ej. `192.168.100.2`).

- `<IP_GATEWAY>`: Dirección IP de la puerta de enlace de la red (ej. `192.168.100.254`).

- `<IP_DNS>`: Servidor de resolución de nombres (ej. `192.168.100.1`).

- `<RED_LOCAL>`: El segmento de red permitido para el retransmisor (relay) local (ej. `192.168.100.0/24`).

- `<USUARIO_CORREO>`: Nombre del usuario del sistema que tendrá un buzón de correo (ej. `juanperez`).

- `<CA_USER>@<CA_HOST>`: Usuario y servidor donde reside la Autoridad Certificadora para copiar los certificados.


# 2. Preparación del Sistema y Red

**Acción:** Configurar el nombre de host corto del servidor.

Bash

```
hostnamectl set-hostname <HOSTNAME_CORREO>
```

**Acción:** Ajustar la resolución local para incluir el FQDN apuntando a la interfaz de loopback.
```
nano /etc/hosts
	127.0.0.1       localhost
	127.0.1.1       <FQDN_CORREO>    <HOSTNAME_CORREO>
```

**Acción:** Configurar los servidores DNS locales.
```
nano /etc/resolv.conf 
	search <DOMINIO_PRINCIPAL> 
	nameserver <IP_DNS> 
```

**Acción:** Asignar la dirección IP estática en la interfaz de red primaria. _(Nota: Ajusta `enp0s3` según el nombre real de tu interfaz)._
```
nano /etc/network/interfaces
	# The primary network interface
	allow-hotplug enp0s3
	iface enp0s3 inet static
	    address <IP_SERVIDOR>
	    netmask 255.255.255.0
	    gateway <IP_GATEWAY>
```

**Acción:** Actualizar los repositorios y paquetes del sistema.
```
apt update && apt upgrade -y
```

# 3. Gestión de Certificados SSL/TLS

**Acción:** Importar los certificados TLS/SSL generados desde el servidor CA y asignar los permisos restrictivos necesarios.

Copiar certificados desde el servidor CA
```
scp <CA_USER>@<CA_HOST>:/ruta/ca/certs/<HOSTNAME_CORREO>.crt /etc/ssl/certs/<HOSTNAME_CORREO>.crt
scp <CA_USER>@<CA_HOST>:/ruta/ca/private/<HOSTNAME_CORREO>.key /etc/ssl/private/<HOSTNAME_CORREO>.key
scp <CA_USER>@<CA_HOST>:/ruta/ca/certs/ca-root.crt /etc/ssl/certs/ca-root.crt
```

Asegurar la llave privada
```
chmod 600 /etc/ssl/private/<HOSTNAME_CORREO>.key
chown root:root /etc/ssl/private/<HOSTNAME_CORREO>.key
```

# 4. Instalación y Configuración de Postfix (MTA)

**Acción:** Instalar el agente de transferencia de correo y herramientas básicas.
```
apt update && apt install -y postfix mailutils
```

**Acción:** Respaldar y optimizar la configuración principal (`main.cf`).
[[main.cf|main.cf]]

**Acción:** Respaldar y configurar el proceso maestro (`master.cf`) para habilitar el puerto de envío seguro SMTP Submission (587).
```
cp /etc/postfix/master.cf /etc/postfix/master.cf.bak
nano /etc/postfix/master.cf
```
[[master.cf|master.cf]]

# 5. Instalación y Configuración de Dovecot (IMAP/POP3)

**Acción:** Instalar los servicios de Dovecot.
```
apt install -y dovecot-imapd dovecot-pop3d
```

**Acción:** Respaldar y configurar los parámetros SSL (`10-ssl.conf`) para forzar conexiones seguras.
```
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.bak
nano /etc/dovecot/conf.d/10-ssl.conf
```
[[10-ssl.conf|10-ssl.conf]]

**Acción:** Respaldar y configurar los mecanismos de autenticación (`10-auth.conf`).
```
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.bak
nano /etc/dovecot/conf.d/10-auth.conf
```
[[10-auth.conf|10-auth.conf]]

**Acción:** Respaldar y configurar los sockets de autenticación para Postfix (`10-master.conf`).
```
cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.bak
nano /etc/dovecot/conf.d/10-master.conf
```
[[10-master.conf|10-master.conf]]

## 6. Creación de Usuarios y Estructura Maildir

**Acción:** Crear usuarios del sistema sin acceso a shell y preparar su estructura de buzón Maildir.
Crear usuario
```
useradd -m -s /sbin/nologin -c "Usuario de Correo" <USUARIO_CORREO>
passwd <USUARIO_CORREO>
```
Crear estructura Maildir y asignar permisos
```
mkdir -p /home/<USUARIO_CORREO>/Maildir/{new,cur,tmp}
chown -R <USUARIO_CORREO>:<USUARIO_CORREO> /home/<USUARIO_CORREO>/Maildir/
chmod -R 700 /home/<USUARIO_CORREO>/Maildir/
```

## 7. Gestión, Verificación y Troubleshooting

**Control de Servicios:**
```
systemctl restart postfix dovecot
systemctl enable postfix dovecot
systemctl status postfix dovecot
```

**Verificación de Configuración Activa:** Para validar que no hay errores de sintaxis y ver los parámetros cargados en memoria:
Verificar Postfix
```
postconf -n
```
Verificar Dovecot
```
doveconf -n
```

**Prueba de Conexión TLS/SSL:** Comprueba que el certificado se esté entregando correctamente en el puerto IMAPS (993).
```
openssl s_client -connect <FQDN_CORREO>:993 -CAfile /etc/ssl/certs/ca-root.crt
```

**Diagnóstico de Correo (Troubleshooting):**

- **Ver la cola de correos:** `mailq` o `postqueue -p`

- **Forzar el envío de la cola:** `postqueue -f`

- **Borrar todos los correos encolados:** `postsuper -d ALL`

- **Monitoreo en tiempo real de transacciones (Logs):**
```
tail -f /var/log/mail.log
```

Alternativa moderna si el sistema usa systemd-journald:
```
journalctl -u postfix -u dovecot -f
```

- **Fallo de Autenticación (SASL):** Si los clientes no pueden enviar correo (Error 535), verifica que el socket en `/var/spool/postfix/private/auth` se creó correctamente y tiene los permisos adecuados definidos en `10-master.conf`.

- **Rechazo de Relay (Relay Access Denied):** Verifica que la IP pública o local del cliente esté contemplada en la directiva `mynetworks` dentro de `main.cf`, o que el cliente esté enviando correo autenticado correctamente por el puerto 587.


## 8. Funciones Avanzadas y Próximos Pasos

Una vez validada la funcionalidad base, se recomienda implementar las siguientes mejoras para asegurar un entorno de producción óptimo:

- **Registros DNS de Seguridad:** Implementación de **SPF** (Sender Policy Framework), **DKIM** (DomainKeys Identified Mail) y **DMARC** (Domain-based Message Authentication, Reporting, and Conformance) para evitar que los correos sean marcados como SPAM.

- **Integración Webmail:** Despliegue de un cliente web como **Roundcube** acoplado a un servidor web (Nginx/Apache) y PHP-FPM.

- **Usuarios Virtuales (Base de Datos):** Migración de cuentas de sistema (`/etc/passwd`) a usuarios virtuales gestionados mediante MariaDB/MySQL o PostgreSQL, permitiendo hospedar múltiples dominios virtuales (Virtual Domains).

- **Antispam y Antivirus:** Integración de **SpamAssassin** y **ClamAV** a través de Amavis o Rspamd.