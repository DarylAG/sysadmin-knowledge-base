# Configuración de Squid Proxy Transparente con SSL Bumping

## Convenciones y Variables

Antes de iniciar, identifica y sustituye las siguientes variables en los comandos y archivos de configuración según tu entorno:

- `<HOSTNAME>`: Nombre del servidor proxy (ej. `proxy-server`).
- `<RED_LOCAL>`: Rango de tu red local en formato CIDR (ej. `192.168.100.0/24`).    
- `<PUERTO_DIRECTO>`: Puerto de escucha para peticiones directas, por defecto `3127`.
- `<PUERTO_HTTP_INTERCEPT>`: Puerto de intercepción transparente HTTP, por defecto `3128`.
- `<PUERTO_HTTPS_INTERCEPT>`: Puerto de intercepción transparente HTTPS, por defecto `3129`.

---

## Fase 1: Preparación del Sistema e Instalación

Configura el nombre del host y actualiza el sistema antes de instalar las dependencias necesarias.

Establecer el nombre de host:
```bash
hostnamectl set-hostname <HOSTNAME>
````

Actualizar repositorios e instalar Squid con soporte SSL y utilidades de firewall:

Bash

```
apt update && apt upgrade -y
apt install squid squid-openssl iptables -y
```

## Fase 2: Gestión de Certificados para SSL Bumping

Para que Squid intercepte tráfico cifrado (HTTPS), debe actuar como una Autoridad Certificadora (CA) y generar certificados al vuelo. Se requiere consolidar el certificado público de la CA y su clave privada en un único archivo unificado (`squid.pem`).

1. Crear el directorio exclusivo para los certificados del proxy:
    

Bash

```
mkdir -p /etc/squid/certs
cd /etc/squid/certs/
```

2. Generar el archivo `squid.pem` concatenando el certificado CA (`.crt`) y su llave privada (`.key`):
    

Bash

```
cat ca.crt ca.key > /etc/squid/certs/squid.pem
```

3. Ajustar permisos de seguridad estrictos (imprescindible para el demonio de Squid):
    

Bash

```
chown -R proxy:proxy /etc/squid/certs
chmod 755 /etc/squid/certs
chmod 600 /etc/squid/certs/squid.pem
```

## Fase 3: Inicialización de la Base de Datos SSL

Squid requiere un almacén de memoria y disco para alojar los certificados generados dinámicamente.

1. Crear la estructura de directorios:
    

Bash

```
mkdir -p /var/lib/squid
chown -R proxy:proxy /var/lib/squid
```

2. Inicializar la base de datos `ssl_db` (tamaño ajustado a 4 MB):
    

Bash

```
/usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB
chown -R proxy:proxy /var/lib/squid/ssl_db
```

## Fase 4: Configuración de Squid (`squid.conf`)

Respalda el archivo de configuración original:

Bash

```
mkdir -p /etc/squid/backup
cp /etc/squid/squid.conf /etc/squid/backup/squid.conf.bak
```

Estructura del archivo de configuración optimizado (`/etc/squid/squid.conf`):

Bash

```
# ----------------------------------------------------------------------
# DEFINICIÓN DE REDES Y PUERTOS SEGUROS
# ----------------------------------------------------------------------
acl localnet src <RED_LOCAL>
acl Safe_ports port 80          # http
acl Safe_ports port 443         # https
acl SSL_ports port 443          # https exclusivo

# Control estricto de puertos no estándar
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports

# ----------------------------------------------------------------------
# REGLAS DE CONTROL DE ACCESO (HTTP_ACCESS)
# ----------------------------------------------------------------------
http_access allow localhost
http_access allow localnet
http_access deny all

# ----------------------------------------------------------------------
# PUERTOS Y SSL BUMPING (PROXY TRANSPARENTE)
# ----------------------------------------------------------------------
# Puerto directo
http_port <PUERTO_DIRECTO>

# Intercepción HTTP transparente
http_port <PUERTO_HTTP_INTERCEPT> intercept

# Intercepción HTTPS transparente con inspección de certificados
https_port <PUERTO_HTTPS_INTERCEPT> intercept ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/certs/squid.pem

# Programa de gestión de base de datos SSL
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 4MB

# Fases de inspección SSL (Peeled / Bumped)
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# ----------------------------------------------------------------------
# PARÁMETROS DE SISTEMA
# ----------------------------------------------------------------------
maximum_object_size 4 MB
visible_hostname <HOSTNAME>
shutdown_lifetime 3 seconds
```

## Fase 5: Reglas de Red y Firewall (Iptables)

Redirige el tráfico web saliente de la red local hacia los puertos de intercepción de Squid.

Bash

```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port <PUERTO_HTTP_INTERCEPT>
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port <PUERTO_HTTPS_INTERCEPT>
```

## Gestión, Verificación y Troubleshooting

**1. Verificación de Sintaxis y Arranque:**

Bash

```
# Validar sintaxis antes de reiniciar
squid -k parse

# Reiniciar y habilitar el servicio
systemctl restart squid
systemctl enable squid
```

**2. Monitoreo de Registros en Tiempo Real:**

Bash

```
tail -f /var/log/squid/access.log /var/log/squid/cache.log
```

**3. Corrupción de la Base de Datos SSL (`ssl_db`):** Si el servicio falla por problemas en la generación de certificados dinámicos, reinstancia la base de datos con:

Bash

```
systemctl stop squid
rm -rf /var/lib/squid/ssl_db/*
/usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB
chown -R proxy:proxy /var/lib/squid/ssl_db
systemctl start squid
```