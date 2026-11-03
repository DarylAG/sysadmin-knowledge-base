# Implementación de Servidor NFS (Network File System)

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según tu entorno:
- `<SUBRED>`: Red local autorizada para acceder a los recursos (ej. `192.168.100.0/24`).
- `<USUARIO_VISITA>`: Nombre del usuario restringido (ej. `visita`).
- `<ID_USUARIO>`: UID personalizado para el usuario (ej. `200`).
- `<USUARIO_PERFIL>`: Nombre del usuario asociado al perfil móvil (ej. `alex`).

## 1. Instalación de Paquetes
**Acción:** Instalar el paquete del servidor NFS.
```
apt update && apt install -y nfs-kernel-server
```

## 2. Creación de Usuarios y Directorios
**Acción:** Crear un usuario específico en el sistema para accesos restringidos (ej. visita).
```
useradd -u <ID_USUARIO> -m -s /bin/bash -c "Usuario Visita" <USUARIO_VISITA>
```

**Acción:** Crear la estructura de directorios para los diferentes tipos de recursos compartidos (trabajo, público, protegido y perfiles).
```
mkdir -p /srv/nfs/trabajo_visita
mkdir -p /srv/nfs/publico
mkdir -p /srv/nfs/admin_only
mkdir -p /srv/nfs/homes/<USUARIO_PERFIL>
```

**Acción:** Asignar los propietarios y grupos correspondientes a cada directorio para controlar el acceso a nivel local.
```
chown -R <USUARIO_VISITA>:<USUARIO_VISITA> /srv/nfs/trabajo_visita
chown -R nobody:nogroup /srv/nfs/publico
```

## 3. Configuración de Exportaciones

**Acción:** Editar el archivo principal de NFS para declarar qué carpetas se comparten, con quién y con qué nivel de privilegios.
```
nano /etc/exports
```

```
# /etc/exports
# Definición de recursos compartidos por NFS para los equipos cliente

# 1. Ejemplo para que solamente el usuario específico ("visita") pueda trabajar
# Mantiene los permisos del usuario creador (no hace squash)
/srv/nfs/trabajo_visita    <SUBRED>(rw,sync,no_subtree_check)

# 2. Ejemplo para acceso público a todos los usuarios
# "all_squash" mapea todas las peticiones (sean de quien sean) al usuario anónimo "nobody"
/srv/nfs/publico           <SUBRED>(rw,sync,all_squash,no_subtree_check)

# 3. Ejemplo para proteger el servidor de los usuarios root en los equipos clientes
# "root_squash" (por defecto) convierte las peticiones del root cliente en un usuario sin privilegios
/srv/nfs/admin_only        <SUBRED>(rw,sync,root_squash,no_subtree_check)

# 4. Ejemplo para el concepto de "perfil móvil"
# "no_root_squash" permite que si el cliente es root, también actúe como root en esta carpeta compartida
/srv/nfs/homes/<USUARIO_PERFIL>    <SUBRED>(rw,sync,no_root_squash,no_subtree_check)
```

## 4. Aplicar Cambios y Reiniciar

**Acción:** Exportar (habilitar) los directorios leídos del archivo de configuración y recargar los demonios del sistema.
```
exportfs -arv
systemctl daemon-reload
systemctl restart nfs-kernel-server
```

## 5. Gestión, Verificación y Troubleshooting

- **Verificar exportaciones activas en el servidor:**

```
exportfs -v
```

- **Revisar qué directorios está compartiendo el servidor (útil para diagnosticar desde el cliente):**

```
showmount -e localhost
```

- **Comprobar el estado del servicio y revisar logs:**
```
systemctl status nfs-kernel-server
journalctl -u nfs-kernel-server -f
```

- **Error común (Access Denied / Permission Denied):** Ocurre si la IP del equipo cliente no entra en el rango de la `<SUBRED>` definida en `/etc/exports`, o si los permisos locales de Linux (`chown`/`chmod`) bloquean la escritura a pesar de que NFS la permite.

## Funciones Avanzadas y Próximos Pasos

- **Seguridad con Kerberos:** Migrar a NFSv4 con seguridad `sec=krb5` para requerir autenticación cifrada por usuario, en lugar de confiar ciegamente en la dirección IP del cliente.

- **Montaje automático en clientes (Autofs):** Configurar los equipos clientes para que monten las carpetas NFS dinámicamente solo en el momento en que un usuario intente acceder a ellas, ahorrando recursos de red.