# Standard Operating Procedure (SOP) - Certificate Authority (CA) with OpenSSL

## Overview and Laboratory Topology

This Standard Operating Procedure (SOP) outlines the deployment of an internal **Certificate Authority (CA) using OpenSSL** within a simulated multi-node network infrastructure in GNS3. The CA will provide cryptographic trust for the internal domain services, including generating a specialized unencrypted certificate bundle for transparent SSL inspection (SSL-Bump) on the Squid proxy.

In this environment, all servers and client hosts are interconnected through a central switch (`Switch1`) and routed to the external network/Internet via router `R1` and a `NAT1` cloud interface:

* **Topology Naming Scheme & Static IPs:**
  * `alpha-DDNS-1`: DNS & DHCP Server (`.1`)
  * `beta-CORREO-1`: Mail Server (`.2`)
  * `gamma-WEB&FTP-1`: Web & FTP Server (`.3`)
  * `iota-PROXY-1`: Squid Proxy Server (`.4`)
  * **`delta-CA-1`**: Internal Certificate Authority (CA) - *Primary target of this SOP*
  * `pcw01-1` / `pcl01-1`: Windows and Linux Client Workstations

<p align="center">
  <img src="../Assets/GNS3_LAB.png" alt="GNS3 Lab Topology" width="80%">
</p>

Since the CA server is deployed directly into this pre-addressed topology, the initial system setup focuses on establishing a secure directory structure and defining strict cryptographic policies (`openssl.cnf`) prior to generating the Root CA and issuing domain service certificates.

---

## Conventions and Environment Variables

To ensure the adaptability of this template to any environment, static values have been replaced with placeholders. Before executing the commands, replace the following variables with the actual values for your infrastructure:

**Environment and Naming Variables**
* `<CA_DIR>`: Base path for the CA directory (e.g., `/root/ca`).
* `<CA_NAME>`: Base filename for the CA (e.g., `delta-root-ca`).
* `<NODE_NAME>`: Name of the service or server to be certified (e.g., `iota-proxy`).
* `<NODE_IP>`: Server IP address for the SAN extension (e.g., `192.168.106.4`).

**Identity Variables (X.509 Subject Fields)**
* `<COUNTRY>` *(C - Country)*: 2-letter country code under the ISO 3166-1 alpha-2 standard (e.g., `CR` for Costa Rica).
* `<STATE>` *(ST - State or Province)*: Full name of the state, province, or region (e.g., `Alajuela`).
* `<LOCALITY>` *(L - Locality)*: Name of the city or municipality.
* `<ORGANIZATION>` *(O - Organization)*: Legal name of the company or operating entity.
* `<DOMAIN_NAME>` *(CN - Common Name)*: The primary Fully Qualified Domain Name (FQDN) of the server or the descriptive name of the CA (e.g., `delta.xeon.cr` or `Xeon Root CA`).

---

## 1. Environment Preparation and Directory Structure

Establish the secure directory structure required by OpenSSL to manage the certificate database, serial numbers, and store cryptographic keys with appropriate permissions.

Change the CA server hostname (optional but recommended for identification).
```bash
hostnamectl set-hostname ca-server
```

Create the directory structure.
```bash
mkdir -p <CA_DIR>/{private,certs,newcerts,csr}
```

Restrict access to the private keys folder (read/write only for the owner).
```bash
chmod -v 700 <CA_DIR>/private
```

Initialize the certificate database and the hexadecimal serial number file.
```bash
touch <CA_DIR>/index
openssl rand -hex 16 > <CA_DIR>/serial
```

## 2. CA Configuration File (`root-ca.conf`)

Create the `<CA_DIR>/root-ca.conf` file with the following optimized content. This file defines the signing policies and the cryptographic extensions required for the CA and the issued certificates.
```bash
nano <CA_DIR>/root-ca.conf
```

```conf
[ ca ]
# Define the default section for the 'ca' command
default_ca = CA_default

[ CA_default ]
# Directory structure paths
dir           = <CA_DIR>
certs         = $dir/certs
new_certs_dir = $dir/newcerts
database      = $dir/index
serial        = $dir/serial
RANDFILE      = $dir/private/.rand

# CA Files (Private key and Public certificate)
certificate   = $dir/certs/<CA_NAME>.crt
private_key   = $dir/private/<CA_NAME>.key

# Cryptographic and issuance parameters
default_md    = sha256
name_opt      = ca_default
cert_opt      = ca_default
default_days  = 365
preserve      = no
policy        = policy_strict

[ policy_strict ]
# Subject field requirements for the CA to sign the certificate
countryName             = supplied
stateOrProvinceName     = supplied
organizationName        = supplied
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Parameters for Certificate Signing Requests (CSR) generation
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
# Interactive field template (Ignored if prompt=no is used in the CSR)
countryName         = Country Name (2 letter code)
stateOrProvinceName = State or Province Name
localityName        = Locality Name
organizationName    = Organization Name
commonName          = Common Name

[ v3_ca ]
# Mandatory extensions for a Certificate Authority certificate
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:TRUE
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
# Standard extensions for end-entity server certificates
basicConstraints       = CA:FALSE
nsCertType             = server
nsComment              = "Digital Certificate for Internal Server"
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage               = critical, digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth
```
## 3. Root CA Key and Certificate Generation

Generate the CA private key using AES-256 encryption and a 4096-bit size for maximum security. Then, restrict the file permissions.
```bash
cd <CA_DIR>
openssl genrsa -aes256 -out <CA_DIR>/private/<CA_NAME>.key 4096
chmod 400 <CA_DIR>/private/<CA_NAME>.key
```

Generate the self-signed root certificate (Root CA) using the defined configuration. An extended validity is set (e.g., 3650 days / 10 years), and the `v3_ca` extension is applied to enable its functions as a certificate authority.
```bash
openssl req -config <CA_DIR>/root-ca.conf \
  -key <CA_DIR>/private/<CA_NAME>.key \
  -new -x509 -days 3650 -sha256 -extensions v3_ca \
  -out <CA_DIR>/certs/<CA_NAME>.crt
```

> **CRITICAL REQUIREMENT NOTE: Unencrypted PEM Certificate for Squid Proxy (SSL-Bump)**
> 
> For SSL/TLS inspection (SSL Bumping) on a transparent Squid Proxy, the service requires an unencrypted CA private key to dynamically sign certificates in memory. Furthermore, Squid typically requires the private key and certificate combined into a single `.pem` file.
> 
> Extract an unencrypted copy of the CA private key (you will be prompted for the original key's passphrase):
> ```bash
> openssl rsa -in <CA_DIR>/private/<CA_NAME>.key -out <CA_DIR>/private/<CA_NAME>-nopass.key
> 
> # Strictly protect the unencrypted key
> chmod 400 <CA_DIR>/private/<CA_NAME>-nopass.key
> ```
> 
> Concatenate the unencrypted private key and the public CA certificate into a unified `.pem` file:
> ```bash
> cat <CA_DIR>/private/<CA_NAME>-nopass.key <CA_DIR>/certs/<CA_NAME>.crt > <CA_DIR>/certs/<CA_NAME>-squid-bundle.pem
> 
> chmod 400 <CA_DIR>/certs/<CA_NAME>-squid-bundle.pem
> ```
## 4. Server Certificate Generation (Nodes)

This process must be repeated for each service (Web, Mail, Proxy, etc.) by adapting the variables.

### 4.1. Node CSR Configuration File (`<NODE_NAME>-csr.conf`)

Create a specific configuration file for the node in `<CA_DIR>/csr/<NODE_NAME>-csr.conf`. This approach injects the Subject Alternative Name (SAN) extension directly into the request, which is a strict requirement for modern browsers.

```conf
[ req ]
default_bits       = 2048
default_md         = sha256
distinguished_name = req_distinguished_name
req_extensions     = req_ext
prompt             = no

[ req_distinguished_name ]
# Replace with actual variables
countryName            = <COUNTRY>
stateOrProvinceName    = <STATE>
localityName           = <LOCALITY>
organizationName       = <ORGANIZATION>
commonName             = <DOMAIN_NAME>

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
# Subject Alternative Names (FQDNs, short hostnames, and IP addresses)
DNS.1 = <DOMAIN_NAME>
DNS.2 = www.<DOMAIN_NAME>
DNS.3 = <NODE_NAME>
IP.1  = <NODE_IP>
```
### 4.2. Node Certificate Creation and Signing

Generate the 2048-bit private key for the target server.
```bash
openssl genrsa -out <CA_DIR>/private/<NODE_NAME>.key 2048
```

Generate the Certificate Signing Request (CSR) using the customized configuration file that contains the SAN extensions.
```bash
openssl req -new -key <CA_DIR>/private/<NODE_NAME>.key -sha256 \
  -out <CA_DIR>/csr/<NODE_NAME>.csr \
  -config <CA_DIR>/csr/<NODE_NAME>-csr.conf
```

Sign the node certificate using the Root CA. The SAN extensions (`req_ext`) are explicitly included by passing them directly from the node's `.conf` file via `-extfile`.
```bash
openssl ca -config <CA_DIR>/root-ca.conf -notext \
  -in <CA_DIR>/csr/<NODE_NAME>.csr \
  -out <CA_DIR>/certs/<NODE_NAME>.crt \
  -extensions req_ext \
  -extfile <CA_DIR>/csr/<NODE_NAME>-csr.conf -batch
```

## 5. Verification, Management, and Troubleshooting

### Verification Commands

To audit the correct generation of cryptographic elements, use the following inspection commands:

Inspect a Public Certificate (`.crt` / `.pem`):
```bash
openssl x509 -text -noout -in <CA_DIR>/certs/<NODE_NAME>.crt
```

Inspect a Certificate Signing Request (CSR):
```bash
openssl req -text -noout -verify -in <CA_DIR>/csr/<NODE_NAME>.csr
```

Verify Specific Subject Alternative Name (SAN) Extensions:
```bash
openssl x509 -text -noout -in <CA_DIR>/certs/<NODE_NAME>.crt | grep -A 2 "Subject Alternative Name"
```

Verify Key and Certificate Mathematical Integrity (Matching Modulus Hash):
```bash
openssl rsa -noout -modulus -in <CA_DIR>/private/<NODE_NAME>.key | openssl md5
openssl x509 -noout -modulus -in <CA_DIR>/certs/<NODE_NAME>.crt | openssl md5
```

### Common Troubleshooting

- **Browser Error `ERR_CERT_COMMON_NAME_INVALID`:** Chrome and modern browsers ignore the _Common Name (CN)_ field. If this error appears, the certificate was signed without the _Subject Alternative Name (SAN)_ extension. Verify Step 4.2 and ensure the `-extfile` flag is included during signing.

- **Browser Error `ERR_CERT_AUTHORITY_INVALID`:** The server certificate was successfully validated, but the client system or browser does not trust the issuer. Install `ca.crt` into the client's "Trusted Root Certification Authorities" store.

- **OpenSSL Error `TXT_DB error number 2`:** Indicates an attempt to sign a certificate with a _Subject_ identical to a previously signed entry. Revoke the old entry or modify `unique_subject = no` in the `index.attr` configuration.


## 6. Advanced Features and Next Steps

To elevate this Root CA deployment to an enterprise-grade model, consider implementing the following enhancements:

- **Intermediate CAs:** Follow security best practices by keeping the Root CA offline. Generate an Intermediate CA signed by the Root CA to handle day-to-day certificate issuance.

- **Certificate Revocation Lists (CRL):** Configure and publish periodic `.crl` files to notify clients of revoked or compromised certificates before expiration.

- **Online Certificate Status Protocol (OCSP):** Deploy an OCSP responder to provide real-time certificate validation, reducing client overhead compared to CRL downloads.

- **Automated Distribution:** Package and deploy the public Root CA certificate using Active Directory Group Policies (GPO) or Ansible to automatically populate corporate client trust stores.