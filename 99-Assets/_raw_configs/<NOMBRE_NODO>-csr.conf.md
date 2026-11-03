```bash
[ req ]
default_bits       = 2048
default_md         = sha256
distinguished_name = req_distinguished_name
req_extensions     = req_ext
prompt             = no

[ req_distinguished_name ]
# Reemplazar con variables reales
countryName            = <PAIS>
stateOrProvinceName    = <ESTADO>
localityName           = <LOCALIDAD>
organizationName       = <ORGANIZACION>
commonName             = <DOMINIO>

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
# Nombres alternativos (FQDN, hostnames cortos y Direcciones IP)
DNS.1 = <DOMINIO>
DNS.2 = www.<DOMINIO>
DNS.3 = <NOMBRE_NODO>
IP.1  = <IP_NODO>
```