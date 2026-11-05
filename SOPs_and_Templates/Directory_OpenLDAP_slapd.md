# OpenLDAP - slapd

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según la estructura de tu dominio:
- `<DC1>` y `<DC2>`: Partes de tu nombre de dominio (ej. si el dominio es `dominio.com`, usa `dc=dominio` y `dc=com`).
- `<NOMBRE_GRUPO>`: Nombre del grupo posix (ej. `grupo_ventas`).
- `<ID_GRUPO>`: GID numérico para el grupo (ej. `10000`).
- `<USUARIO_LDAP>`: Nombre de usuario de inicio de sesión (ej. `juan`).
- `<NOMBRE_PILA>` / `<APELLIDO>`: Nombres reales (ej. `Juan` / `Perez`).
- `<NOMBRE_COMPLETO>`: Nombre y apellido juntos (ej. `Juan Perez`).
- `<ID_USUARIO>`: UID numérico del usuario (ej. `10000`).
- `<HASH_PASSWORD>`: Contraseña encriptada generada por OpenLDAP.

## 1. Instalación y Configuración Base del Servidor
**Acción:** Instalar el servidor OpenLDAP e iniciar el asistente de reconfiguración para establecer el dominio base y la contraseña de administrador (`admin`).
```
apt update
apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```

## 2. Creación de la Estructura Base (Unidades Organizativas)
**Acción:** Crear un archivo LDIF para definir las Unidades Organizativas (OUs) principales: Usuarios (`People`) y Grupos (`Groups`).
```
nano unidades.ldif
```

```
# Archivo donde se definen las UO (Unidades Organizativas) del directorio
# Nota: Siempre dejar una línea en blanco al final de cada definición de objeto

dn: ou=People,dc=<DC1>,dc=<DC2>
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=<DC1>,dc=<DC2>
objectClass: organizationalUnit
ou: Groups
```

**Acción:** Inyectar las unidades organizativas a la base de datos LDAP usando el usuario administrador.
```
ldapadd -x -D "cn=admin,dc=<DC1>,dc=<DC2>" -W -f unidades.ldif
```

## 3. Creación de Grupos y Usuarios

**Acción:** Crear el archivo para definir un grupo.
```
nano grupo.ldif
```

```
# Definición del grupo posix al cual pertenecerán los usuarios

dn: cn=<NOMBRE_GRUPO>,ou=Groups,dc=<DC1>,dc=<DC2>
objectClass: posixGroup
cn: <NOMBRE_GRUPO>
gidNumber: <ID_GRUPO>
```

**Acción:** Inyectar el grupo al directorio.
```
ldapadd -x -D "cn=admin,dc=<DC1>,dc=<DC2>" -W -f grupo.ldif
```

**Acción:** Generar una contraseña encriptada para el nuevo usuario. Copia el hash resultante (ej. `{SSHA}xxxxx...`) para usarlo en el siguiente paso.

```
slappasswd
```

**Acción:** Crear el archivo de definición del usuario.
```
nano usuario.ldif
```

```
# Definición del objeto usuario posix

dn: uid=<USUARIO_LDAP>,ou=People,dc=<DC1>,dc=<DC2>
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: <USUARIO_LDAP>
sn: <APELLIDO>
givenName: <NOMBRE_PILA>
cn: <NOMBRE_COMPLETO>
uidNumber: <ID_USUARIO>
gidNumber: <ID_GRUPO>
homeDirectory: /home/<USUARIO_LDAP>
loginShell: /bin/bash
# Reemplazar con el hash exacto generado por slappasswd
userPassword: <HASH_PASSWORD>
```

**Acción:** Inyectar el usuario al directorio.
```
ldapadd -x -D "cn=admin,dc=<DC1>,dc=<DC2>" -W -f usuario.ldif
```

## 4. Validación del Servidor
**Acción:** Comprobar la integridad de la base de datos, reiniciar el servicio y hacer búsquedas de prueba.
```
slaptest -u 
systemctl restart slapd

# Buscar todo el árbol del dominio
ldapsearch -x -b "dc=<DC1>,dc=<DC2>"

# Buscar un usuario específico dentro de la unidad People
ldapsearch -x -b "ou=People,dc=<DC1>,dc=<DC2>" "(uid=<USUARIO_LDAP>)"
```

## 5. Configuración del Cliente Linux (Autenticación Remota)

**Acción:** Instalar las dependencias en el equipo **cliente** que se conectará al servidor LDAP.
```
apt install -y libnss-ldap libpam-ldap ldap-utils nscd 
```

**Acción:** Editar el archivo de resolución de nombres de sistema para incluir `ldap`.
_(Agrega "ldap" al final de las líneas de passwd, group y shadow)_.
```
nano /etc/nsswitch.conf
```

**Acción:** Habilitar la creación automática del directorio "home" cuando el usuario inicie sesión por primera vez en el cliente.
```
pam-auth-update
```

**Acción:** Verificar que el cliente reconoce al usuario de LDAP sin necesidad de iniciar sesión aún.
```
getent passwd <USUARIO_LDAP>
```

## Funciones Avanzadas y Próximos Pasos

- **LDAP Account Manager (LAM):** Puedes instalar una interfaz gráfica web en el servidor para administrar usuarios más fácilmente:
```
apt install ldap-account-manager
```

- **Integración LDAPS:** Habilitar cifrado TLS (puerto 636) usando certificados SSL para proteger el tráfico de contraseñas por la red.
