# Implementación de Servidor FTP (vsftpd)

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según tu entorno:
- `<USUARIO_FTP>`: Nombre del usuario virtual para FTP (ej. `ftpuser`).
- `<GRUPO_FTP>`: Nombre del grupo asignado a los usuarios de FTP (ej. `grupoftp`).
- `<RUTA_PRIVADA>`: Directorio de acceso restringido para los usuarios locales (ej. `/srv/privado`).
- `<RUTA_PUBLICA>`: Directorio de acceso anónimo (ej. `/srv/ftp`).

## 1. Instalación
**Acción:** Actualizar los repositorios e instalar el demonio del servidor FTP (vsftpd).
```
apt update && apt upgrade -y
apt install -y vsftpd
```

## 2. Configuración Principal del Servidor
**Acción:** Editar el archivo de configuración de vsftpd. Debes buscar y habilitar (quitar el `#`) las opciones básicas, y luego añadir los parámetros personalizados al final del archivo.
```
nano /etc/vsftpd.conf
```

```
# =====================================================================
# 1. BUSCAR Y VERIFICAR/DESCOMENTAR ESTAS LÍNEAS EXISTENTES
# =====================================================================
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022

# =====================================================================
# 2. AÑADIR AL FINAL DEL ARCHIVO (Reglas personalizadas)
# =====================================================================
# Permite acceso anónimo sin pedir contraseña
no_anon_password=YES

# Define el directorio raíz para los usuarios anónimos
anon_root=<RUTA_PUBLICA>

# Deshabilita la subida de archivos para anónimos (Seguridad)
anon_upload_enable=NO

# Permite autenticación de usuarios con shells inválidas como /bin/false
check_shell=NO

# Define el directorio raíz para los usuarios locales/autenticados
local_root=<RUTA_PRIVADA>
```

## 3. Preparación del Sistema de Usuarios y Directorios

**Acción:** Crear el directorio privado.
```
mkdir -p <RUTA_PRIVADA>
```

**Acción:** Crear el usuario de FTP. Le asignamos la ruta privada como su "home" y establecemos `/bin/false` como su shell para evitar que inicie sesión en la consola del servidor (SSH/TTY).
```
useradd -m -d <RUTA_PRIVADA> -s /bin/false <USUARIO_FTP>
passwd <USUARIO_FTP>
```

**Acción:** Para que vsftpd permita el ingreso a usuarios que usan `/bin/false`, debemos declarar esta shell como válida en el sistema.
```
nano /etc/shells
```

```
# /etc/shells: valid login shells
/bin/sh
/bin/bash
/usr/bin/bash
/bin/rbash
/usr/bin/rbash
/bin/dash
/usr/bin/dash

# Añadir esta línea al final para permitir login a servicios de red sin dar acceso a terminal:
/bin/false
```

## 4. Asignación de Grupos y Permisos

**Acción:** Crear un grupo específico para el FTP y agregar al usuario a dicho grupo.
```
groupadd <GRUPO_FTP>
usermod -aG <GRUPO_FTP> <USUARIO_FTP>
```

**Acción:** Ajustar los propietarios y permisos del directorio privado para que el grupo tenga control, pero el propietario raíz (`root`) mantenga la propiedad principal (requerimiento de seguridad de vsftpd para entornos enjaulados).
```
chown -R root:<GRUPO_FTP> <RUTA_PRIVADA>
chmod -R 775 <RUTA_PRIVADA>
```

## 5. Aplicar Cambios

**Acción:** Reiniciar el servicio para aplicar toda la configuración.
```
systemctl restart vsftpd.service
systemctl enable vsftpd.service
```

## 6. Gestión, Verificación y Troubleshooting

- **Comprobar estado del servicio:** `systemctl status vsftpd`

- **Revisar logs de conexiones FTP:** `journalctl -u vsftpd -f` o verificar `/var/log/vsftpd.log`.

- **Error común (530 Login incorrect):** Ocurre casi siempre por dos razones:
    
    1. Se te olvidó agregar `/bin/false` al final del archivo `/etc/shells`.
    
    2. En el archivo `vsftpd.conf` hace falta la directiva `check_shell=NO`.
    
- **Error común (500 OOPS: vsftpd: refusing to run with writable root inside chroot):** Pasa si le das permisos `777` a la carpeta `<RUTA_PRIVADA>`. vsftpd exige que el directorio raíz del usuario no sea escribible por todos.


## Funciones Avanzadas y Próximos Pasos

- **Cifrado (FTPS):** Configurar certificados SSL/TLS en `vsftpd.conf` para encriptar la transferencia de archivos y las contraseñas, evitando que viajen en texto plano por la red.

- **Enjaulado estricto (chroot_local_user=YES):** Forzar a los usuarios locales a que no puedan navegar más allá de su carpeta asignada.