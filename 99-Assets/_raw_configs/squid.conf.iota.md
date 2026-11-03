# ----------------------------------------------------------------------
# PUERTOS SEGUROS
# ----------------------------------------------------------------------
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http

acl SSL_ports port 443

# Denegar peticiones a puertos que no sean los definidos como seguros
http_access deny !Safe_ports

# Denegar el método CONNECT a puertos que no sean SSL
http_access deny CONNECT !SSL_ports

# ----------------------------------------------------------------------
# DEFINICIÓN DE REDES
# ----------------------------------------------------------------------
# Define la red "iota". Ajusta el rango 192.168.106.0/24 según el
# direccionamiento que estés utilizando en tu topología.
acl iota src 192.168.100.0/24

# ----------------------------------------------------------------------
# REGLAS DE CONTROL DE ACCESO (HTTP_ACCESS)
# ----------------------------------------------------------------------
# Permitir acceso a la máquina local (localhost)
http_access allow localhost

# Permitir acceso a los equipos dentro de la red iota
http_access allow iota

# Denegar por defecto cualquier otra solicitud que no coincida con lo anterior
http_access deny all

# ----------------------------------------------------------------------
# PARÁMETROS DE RED Y CACHÉ
# ----------------------------------------------------------------------
# Puerto en el que escucha el servicio proxy
http_port 3128

# Tamaño máximo de los objetos que se guardarán en caché
maximum_object_size 4 MB