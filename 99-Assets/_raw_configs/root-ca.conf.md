```bash
[ ca ]
# Define la sección por defecto para el comando 'ca'
default_ca = CA_default

[ CA_default ]
# Rutas de la estructura de directorios
dir           = <DIR_CA>
certs         = $dir/certs
new_certs_dir = $dir/newcerts
database      = $dir/index
serial        = $dir/serial
RANDFILE      = $dir/private/.rand

# Archivos de la CA (Llave privada y Certificado público)
certificate   = $dir/certs/<NOMBRE_CA>.crt
private_key   = $dir/private/<NOMBRE_CA>.key

# Parámetros criptográficos y de emisión
default_md    = sha256
name_opt      = ca_default
cert_opt      = ca_default
default_days  = 365
preserve      = no
policy        = policy_strict

[ policy_strict ]
# Requisitos de los campos del Subject para que la CA firme el certificado
countryName             = supplied
stateOrProvinceName     = supplied
organizationName        = supplied
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Parámetros para la generación de solicitudes (CSR)
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
# Plantilla interactiva de campos (Ignorada si se usa prompt=no en el CSR)
countryName         = Country Name (2 letter code)
stateOrProvinceName = State or Province Name
localityName        = Locality Name
organizationName    = Organization Name
commonName          = Common Name

[ v3_ca ]
# Extensiones obligatorias para un certificado de Autoridad Certificadora
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:TRUE
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
# Extensiones estándar para certificados de servidor final
basicConstraints       = CA:FALSE
nsCertType             = server
nsComment              = "Certificado Digital para Servidor Interno"
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth