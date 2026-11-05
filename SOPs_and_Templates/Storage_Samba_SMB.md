# Implementación de Servidor Samba (SMB)

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según tu entorno:
- `<USUARIO_SMB>`: Nombre del usuario principal en Linux y Samba (ej. `daryl`).
- `<USUARIO_SECUNDARIO>`: Nombre de un segundo usuario para probar accesos compartidos.
- `<IP_SERVIDOR>`: Dirección IP de tu servidor Samba (ej. `192.168.1.50`).

## 1. Instalación y Respaldo
**Acción:** Actualizar repositorios, instalar los paquetes de Samba y hacer una copia de seguridad del archivo de configuración original.
```
apt update
apt install -y samba samba-common-bin
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
````

## 2. Creación de Usuarios
**Acción:** Crear usuarios en el sistema Linux (sin acceso a consola/shell por seguridad) y luego asignarles una contraseña específica para Samba.
```
adduser <USUARIO_SMB> --shell /usr/sbin/nologin
adduser <USUARIO_SECUNDARIO> --shell /usr/sbin/nologin

# Te pedirá ingresar la contraseña para Samba
smbpasswd -a <USUARIO_SMB>
smbpasswd -a <USUARIO_SECUNDARIO>
```

## 3. Preparación de Directorios y Permisos Locales
**Acción:** Crear las carpetas que se van a compartir y ajustar los permisos a nivel de sistema operativo (Linux).
```
mkdir -p /srv/samba/publico
mkdir -p /srv/samba/privado
mkdir -p /srv/samba/documentos

# Permisos totales para la carpeta pública y de documentos compartidos
chmod -R 777 /srv/samba/publico
chmod -R 777 /srv/samba/documentos

# Permisos estrictos solo para el dueño en la carpeta privada
chown -R <USUARIO_SMB>:<USUARIO_SMB> /srv/samba/privado
chmod -R 700 /srv/samba/privado
```

_(Nota: Para entornos más seguros en la carpeta "documentos", en lugar de `777`, se recomienda crear un grupo común, agregar ambos usuarios al grupo y usar `chmod 770`)._

## 4. Configuración de Recursos Compartidos
**Acción:** Editar el archivo principal de Samba para declarar las carpetas, sus permisos de red y quién puede acceder a ellas.
```
nano /etc/samba/smb.conf
```

```
# =====================================================================
# RECURSOS COMPARTIDOS (Añadir al final de /etc/samba/smb.conf)
# =====================================================================

# Opción A: Acceso Público (Sin contraseña)
[Publico]
   comment = Carpeta Publica Compartida
   path = /srv/samba/publico
   browseable = yes
   writable = yes
   guest ok = yes
   public = yes
   force create mode = 0666
   force directory mode = 0777

# Opción B: Acceso Privado (Solo un usuario específico)
[Privado_<USUARIO_SMB>]
   comment = Carpeta Privada de <USUARIO_SMB>
   path = /srv/samba/privado
   browseable = yes
   writable = yes
   valid users = <USUARIO_SMB>
   guest ok = no

# Opción C: Acceso Compartido Mixto (Varios usuarios, diferentes permisos)
[Documentos]
   comment = Documentos Compartidos
   path = /srv/samba/documentos
   browseable = yes
   # <USUARIO_SECUNDARIO> solo puede leer
   read list = <USUARIO_SECUNDARIO>
   # <USUARIO_SMB> puede leer y escribir
   write list = <USUARIO_SMB>
   valid users = <USUARIO_SMB>, <USUARIO_SECUNDARIO>
   guest ok = no
```

## 5. Validación y Reinicio

**Acción:** Verificar que no haya errores de sintaxis en el archivo de configuración y reiniciar los demonios de Samba.
```
testparm
systemctl restart smbd nmbd
systemctl enable smbd nmbd
```

## 6. Configuración desde un Cliente Linux

**Acción:** En un equipo cliente con Debian/Ubuntu, instalar las utilidades CIFS y montar la carpeta privada a través de la red.
```
apt install -y cifs-utils
mkdir -p /mnt/samba

# Cambia "Privado_<USUARIO_SMB>" por el nombre real del recurso entre corchetes del smb.conf
mount -t cifs -o username=<USUARIO_SMB> //<IP_SERVIDOR>/Privado_<USUARIO_SMB> /mnt/samba
```

## 7. Gestión, Verificación y Troubleshooting

- **Comprobar estado de los servicios:** `systemctl status smbd nmbd`

- **Revisar conexiones activas al servidor:** `smbstatus`

- **Error común (Access Denied):** Ocurre cuando los permisos en `/etc/samba/smb.conf` permiten el acceso, pero los permisos locales de Linux (`chown`/`chmod` sobre `/srv/samba/...`) lo bloquean. Samba nunca puede dar más permisos de los que el propio sistema de archivos de Linux autoriza.