# Implementación de Autoridad Certificadora (CA) local con OpenSSL

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según los datos de tu organización:
- `<PAIS>`: Código de país de 2 letras (ej. `CR`).
- `<PROVINCIA>`: Estado o Provincia (ej. `Alajuela`).
- `<LOCALIDAD>`: Ciudad o Cantón (ej. `Grecia`).
- `<ORGANIZACION>`: Nombre de la empresa o institución (ej. `INA`).
- `<OU>`: Unidad Organizativa o departamento (ej. `Telematica`).
- `<NOMBRE_CA>`: Nombre común para tu CA Raíz (ej. `CA Raiz INA`).
- `<DOMINIO>`: Tu dominio principal (ej. `ina.com`).
- `<DOMINIO_ALT>`: Dominios alternativos (ej. `tienda.local`).
- `<RUTA_CERTS>`: Ruta donde guardarás los certificados de los servidores (ej. `/etc/ssl/certs_ina`).

## 1. Preparación del Entorno y Base de Datos de la CA
**Acción:** Crear la estructura de directorios necesaria para la CA y establecer permisos estrictos de seguridad.
```
mkdir -p /ca/private /ca/certs /ca/crl /ca/requests /ca/newcerts
chmod 700 /ca
chmod 700 /ca/private
````

**Acción:** Crear el archivo índice (base de datos de certificados emitidos).
```
touch /ca/index.txt
```

**Acción:** Crear el archivo de número de serie inicial para los certificados.
```
nano /ca/serial
```

``` PLAINTEXT
1000
```
## 2. Archivo de Configuración de la CA

**Acción:** Crear el archivo principal que dictará las reglas, vigencia y políticas criptográficas de tu Autoridad Certificadora.
```
nano /ca/openssl.cnf
```

```
# Archivo principal de configuracion de la CA Local

[ ca ]
default_ca = CA_default

[ CA_default ]
dir             = /ca                     # Directorio principal de la infraestructura
certs           = $dir/certs              # Directorio para guardar certificados emitidos
new_certs_dir   = $dir/newcerts           # Copias de seguridad automaticas
database        = $dir/index.txt          # Archivo indice de la CA
serial          = $dir/serial             # Archivo de numeracion serial
private_key     = $dir/private/ca.key.pem # Llave secreta de la CA
certificate     = $dir/certs/ca.cert.pem  # Certificado publico de la CA

default_md      = sha256                  # Algoritmo de hash
default_days    = 3650                    # Validez por defecto (10 años)
preserve        = no                      # No reordenar los nombres de los campos
policy          = policy_strict

[ policy_strict ]
# Datos del solicitante que deben coincidir (match) o no (optional/supplied) con la CA
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied        # El FQDN (ej. www.dominio.com)
emailAddress            = optional

[ req ]
# Seccion para generar las claves
default_bits            = 4096            # Tamaño de la llave RSA
distinguished_name      = req_distinguished_name
string_mask             = utf8only
x509_extensions         = v3_ca

[ req_distinguished_name ]
countryName                     = Pais (2 letras)
countryName_default             = <PAIS>
stateOrProvinceName             = Provincia
stateOrProvinceName_default     = <PROVINCIA>
localityName                    = Localidad
localityName_default            = <LOCALIDAD>
organizationName                = Organizacion
organizationName_default        = <ORGANIZACION>
organizationalUnitName_default  = <OU>
commonName                      = Nombre Comun
commonName_default              = <NOMBRE_CA>

[ v3_ca ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints        = critical, CA:true      # Permite firmar otros certificados
keyUsage                = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
basicConstraints        = CA:FALSE
nsCertType              = server
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid,issuer:always
keyUsage                = critical, digitalSignature, keyEncipherment
extendedKeyUsage        = serverAuth

[ client_cert ]
basicConstraints        = CA:FALSE
nsCertType              = client
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid,issuer:always
keyUsage                = critical, digitalSignature, keyEncipherment
extendedKeyUsage        = clientAuth
```
## 3. Generación del Certificado Raíz (Root CA)

**Acción:** Generar la llave privada y el certificado público autofirmado de tu Autoridad Certificadora (con validez de 10 años, según la configuración).
```
openssl req -config /ca/openssl.cnf -new -x509 -extensions v3_ca -keyout /ca/private/ca.key.pem -out /ca/certs/ca.cert.pem
```
_(Nota: El sistema te pedirá ingresar una frase de paso (passphrase) para proteger la llave privada `ca.key.pem`. Guárdala en un lugar seguro)._

**Acción:** Actualizar el almacén de certificados del sistema (opcional, para que el servidor local confíe en su propia CA).
```
cp /ca/certs/ca.cert.pem /usr/local/share/ca-certificates/ca.crt
update-ca-certificates
```

## 4. Generación de Certificados para Servidores (Llave y CSR)

_Nota: Este proceso se realiza cuando quieres emitir un certificado para un servidor web, correo, etc._

**Acción:** Crear directorio para guardar los certificados del servicio y generar su llave privada de 2048 bits.
```
mkdir -p <RUTA_CERTS>
chmod 700 <RUTA_CERTS>
openssl genrsa -out <RUTA_CERTS>/web.key 2048
```

**Acción:** Generar la Solicitud de Firma de Certificado (CSR) inyectando los Nombres Alternativos del Sujeto (SAN) directamente desde la terminal. _(Reemplaza las variables en el comando antes de ejecutarlo)._
```
openssl req -new -key <RUTA_CERTS>/web.key \
  -subj "/C=<PAIS>/ST=<PROVINCIA>/L=<LOCALIDAD>/O=<ORGANIZACION>/OU=<OU>/CN=www.<DOMINIO>" \
  -addext "subjectAltName = DNS:www.<DOMINIO>, DNS:<DOMINIO>, DNS:www.<DOMINIO_ALT>, DNS:<DOMINIO_ALT>" \
  -out <RUTA_CERTS>/web.csr
```

## 5. Gestión, Verificación y Troubleshooting

- **Verificar la sintaxis del archivo `openssl.cnf` antes de emitir:**
```
openssl req -config /ca/openssl.cnf -new -nodes -keyout /dev/null -out /dev/null
```

- **Verificar el contenido del CSR generado:**
```
openssl req -text -noout -verify -in <RUTA_CERTS>/web.csr
```

- **Verificar el certificado de la CA:**
```
openssl x509 -text -noout -in /ca/certs/ca.cert.pem
```
 
- **Error común (TXT_DB error number 2):** Ocurre si intentas firmar dos certificados con el mismo "Subject" (mismos datos) y en el `openssl.cnf` tienes la directiva `unique_subject = yes` en el archivo `index.txt.attr`.

## Funciones Avanzadas y Próximos Pasos

- **Firma del CSR:** El siguiente paso lógico es que la CA firme la solicitud (`web.csr`) para emitir el `web.crt` definitivo usando `openssl ca ...`.

- **Listas de Revocación (CRL):** Configurar y publicar una lista de certificados revocados para invalidar certificados comprometidos antes de que expiren.