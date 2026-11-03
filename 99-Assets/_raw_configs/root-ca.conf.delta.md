```bash
[ ca ]
# 'man ca'
# Used by the ca command
default_ca = CA_default

[ CA_default ]
# Directory and file locations
dir = /home/administrator/ca
certs = $dir/certs
new_certs_dir = $dir/newcerts
database = $dir/index
serial = $dir/serial
RANDFILE = $dir/private/.rand
# RANDFILE is for storing seed data for random number generation

# Root CA certificate and key locations
certificate = $dir/certs/delta-ca.crt
private_key = $dir/private/delta-ca.key

# Default message digest, we'll opt for SHA2 256bits
default_md = sha256

name_opt = ca_default
cert_opt = ca_default
default_days = 365
preserve = no
policy = policy_strict

[ policy_strict ]
countryName = supplied
stateOrProvinceName = supplied
organizationName = supplied
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

[ req ]
# 'man req'
# Used by the req command
default_bits = 2048
distinguished_name = req_distinguished_name
string_mask = utf8only
default_md = sha256

# Extensions to use for -x509
x509_extensions = server_cert

[ req_distinguished_name ]
# Certificate signing request
countryName = Country Name (2 letter code)
stateOrProvinceName = State or Province Name
localityName = Locality Name
organizationName = Organization Name
organizationalUnitName = Organizational Unit Name
commonName = Common Name
emailAddress = Email Address
# Defaults
countryName_default = CR
stateOrProvinceName_default = Alajuela
organizationName_default = xeon.cr

[ v3_ca ]
# ' man x509v3_config'
# Extensions for root CA
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints  = critical, CA:TRUE
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
# ' man x509v3_config`
# Extensions for client certificates
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "Certificado Digital para Servidores XEON"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage  = serverAuth
```
