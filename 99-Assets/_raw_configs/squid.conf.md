```
# ----------------------------------------------------------------------
# DEFINICIÓN DE REDES Y PUERTOS SEGUROS
# ----------------------------------------------------------------------
acl localnet src <RED_LOCAL>    # Define tu red local
acl Safe_ports port 80          # http
acl Safe_ports port 443         # https
acl SSL_ports port 443          # https exclusivo

# Denegar peticiones y CONNECT a puertos no seguros
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
# Puerto base para Squid 6+ (evita advertencias)
http_port <PUERTO_HTTP>

# Intercepción HTTP transparente
http_port <PUERTO_HTTP_INTERCEPT> intercept

# Intercepción HTTPS transparente con certificados dinámicos
https_port <PUERTO_HTTPS> intercept ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/certs/<NOMBRE_CA>.pem

# Programa de gestión de base de datos SSL
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 4MB

# Reglas de inspección SSL (Paso 1, 2 y 3)
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# ----------------------------------------------------------------------
# PARÁMETROS DE RED, CACHÉ Y SISTEMA
# ----------------------------------------------------------------------
maximum_object_size 4 MB
visible_hostname <HOSTNAME>
```


