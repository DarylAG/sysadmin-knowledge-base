
# Implementación de Servidor Web (Apache2)

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según tu entorno:
- `<DOMINIO>`: Nombre del dominio principal (ej. `sitio.com`).
- `<DOMINIO_SECUNDARIO>`: Nombre para un segundo sitio web (ej. `realbusiness.local`).
- `<PUERTO_ALT>`: Puerto alternativo para el segundo sitio (ej. `8081`).
- `<USUARIO>`: Nombre de un usuario del sistema (ej. `ana` o `luis`) para su propio sitio web.

## 1. Instalación y Configuración Básica (VirtualHost en puerto 80)
**Acción:** Instalar Apache2.
```bash
apt update && apt install -y apache2
```

**Acción:** Crear el archivo de configuración (VirtualHost) para el sitio principal.
```bash
nano /etc/apache2/sites-available/<DOMINIO>.conf
```

```conf
# VirtualHost para el sitio web principal
<VirtualHost *:80> 
    ServerAdmin webmaster@<DOMINIO>
    ServerName <DOMINIO>
    ServerAlias www.<DOMINIO> 
    
    # Ruta donde se almacenan los archivos del sitio
    DocumentRoot /var/www/html/<DOMINIO>
    DirectoryIndex index.html 

    # Archivos de log específicos para este dominio
    ErrorLog ${APACHE_LOG_DIR}/<DOMINIO>_error.log 
    CustomLog ${APACHE_LOG_DIR}/<DOMINIO>_access.log combined 
</VirtualHost>
```

**Acción:** Crear el directorio raíz del sitio y su página de inicio.
```bash
mkdir -p /var/www/html/<DOMINIO>
nano /var/www/html/<DOMINIO>/index.html
```

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Página de Prueba</title>
</head>
<body>
    <h1>Bienvenido a <DOMINIO></h1>
    <p>El servidor web Apache2 está funcionando correctamente.</p>
</body>
</html>
```


**Acción:** Habilitar el nuevo sitio, deshabilitar el sitio por defecto y recargar el servicio.
```bash
a2ensite <DOMINIO>.conf
a2dissite 000-default.conf
systemctl reload apache2
```

## 2. Configuración de Sitio en Puerto Alternativo (VirtualHost en otro puerto)

**Acción:** Modificar el archivo de puertos para que Apache escuche en el nuevo puerto.
```bash
nano /etc/apache2/ports.conf
```

```conf
# /etc/apache2/ports.conf
# Archivo de definición de puertos de escucha para Apache2

Listen 80

# Agregar el puerto alternativo
Listen <PUERTO_ALT>

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>
```

**Acción:** Crear el VirtualHost para el segundo sitio.
```bash
nano /etc/apache2/sites-available/segundo.conf
```

```conf
# VirtualHost para el sitio en puerto alternativo
<VirtualHost *:<PUERTO_ALT>>
    ServerName <DOMINIO_SECUNDARIO>
    # O usa otro nombre si prefieres, ej: segundo.local

    DocumentRoot /var/www/html/segundo
    DirectoryIndex index.html

    ErrorLog ${APACHE_LOG_DIR}/segundo_error.log
    CustomLog ${APACHE_LOG_DIR}/segundo_access.log combined
</VirtualHost>
```

**Acción:** Crear el directorio raíz y la página de prueba, habilitar el sitio y reiniciar.
```bash
mkdir -p /var/www/html/segundo
nano /var/www/html/segundo/index.html
# (Pega un HTML básico que diga "Hola desde puerto alternativo")

a2ensite segundo.conf
systemctl restart apache2
```

## 3. Sitios Web Personales para Usuarios del Sistema (UserDir)

**Acción:** Crear un usuario (si no existe), habilitar el módulo `userdir` y reiniciar el servicio.
```bash
adduser <USUARIO>
a2enmod userdir
systemctl restart apache2
```

**Acción:** Crear la carpeta `public_html` dentro del perfil del usuario, asignar permisos correctos y crear su página web.
```bash
mkdir /home/<USUARIO>/public_html
chown <USUARIO>:<USUARIO> /home/<USUARIO>/public_html
chmod 755 /home/<USUARIO>
nano /home/<USUARIO>/public_html/index.html
```

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Página Personal</title>
</head>
<body>
    <h1>Sitio web personal de <USUARIO></h1>
    <p>Servido a través del módulo UserDir de Apache2.</p>
</body>
</html>
```

## 4. Gestión, Verificación y Troubleshooting

- **Comprobar el estado del servicio:** `systemctl status apache2`

- **Verificar sintaxis de configuración (CRÍTICO antes de recargar):**
```
apache2ctl configtest
```

- **Monitorear logs de acceso y errores en tiempo real:**
```
tail -f /var/log/apache2/error.log
tail -f /var/log/apache2/access.log
```

- **Errores comunes (403 Forbidden):** Suele ocurrir si los permisos en `/var/www/html/` o en `/home/<USUARIO>/` son muy restrictivos (Asegurar que Apache tenga permisos de lectura y ejecución en la ruta).

## Funciones Avanzadas y Próximos Pasos

- **Seguridad TLS/SSL:** Instalar y configurar certificados usando OpenSSL o Let's Encrypt para cifrar el tráfico por HTTPS (puerto 443).

- **Protección de directorios:** Habilitar autenticación básica (`htpasswd`) para restringir el acceso a ciertos sitios web.