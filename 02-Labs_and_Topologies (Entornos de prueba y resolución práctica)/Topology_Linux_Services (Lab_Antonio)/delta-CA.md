
hostnamectl set-hostname delta
mkdir -p ca/{private, certs, newcerts, csr}
chmod -v 700 ca/private

touch ca/index  
openssl rand -hex 16 > ca/serial 

cd ca
openssl genrsa -aes256 -out /home/administrator/ca/private/delta-ca.key 4096
chmod 400 /home/administrator/ca/private/delta-ca.key

openssl rsa -in delta-ca.key -out delta-ca-sinpass.key (Para squidproxy transparente.)

nano [[root-ca.conf.delta]]

openssl req -config /home/administrator/ca/root-ca.conf  -key /home/administrator/ca/private/delta-ca.key  -new -x509 -days 3650 -sha256 -extensions v3_ca  -out /home/administrator/ca/certs/delta-ca.crt

#Certificado_gammaWEB
openssl genrsa -out /home/administrator/ca/private/gamma.key 2048
nano /home/administrator/ca/csr/[[gamma-csr.conf]]

#Generar_Certificado
openssl req -new -key /home/administrator/ca/private/gamma.key -sha256 -out /home/administrator/ca/csr/gamma.csr -config /home/administrator/ca/csr/gamma-csr.conf

#Firmar_Certificado
openssl ca -config /home/administrator/ca/root-ca.conf -notext -in /home/administrator/ca/csr/gamma.csr -out /home/administrator/ca/certs/gamma.crt -extensions req_ext -extfile /home/administrator/ca/csr/gamma-csr.conf -batch

#Certificado_betaCORREO
openssl genrsa -out /home/administrator/ca/private/beta.key 2048
nano /home/administrator/ca/csr/[[beta-csr.conf]]

#Generar_Certificado
openssl req -new -key /home/administrator/ca/private/beta.key -sha256 -out /home/administrator/ca/csr/beta.csr -config /home/administrator/ca/csr/beta-csr.conf

#Firmar_Certificado 
openssl ca -config /home/administrator/ca/root-ca.conf -notext -in /home/administrator/ca/csr/beta.csr -out /home/administrator/ca/certs/beta.crt -extensions req_ext -extfile /home/administrator/ca/csr/beta-csr.conf -batch

#VerificarSAN
openssl x509 -text -noout -in /home/administrator/ca/certs/gamma.crt | grep -A 2 "Subject Alternative Name"

#Certificado_iota-PrOXY
openssl genrsa -out /home/administrator/ca/private/iota.key 2048

nano /home/administrator/ca/csr/[[iota-csr.conf]]

openssl req -new -key /home/administrator/ca/private/iota.key -sha256 -out /home/administrator/ca/csr/iota.csr -config /home/administrator/ca/csr/iota-csr.conf

openssl ca -config /home/administrator/ca/root-ca.conf -notext -in /home/administrator/ca/csr/iota.csr -out /home/administrator/ca/certs/iota.crt -extensions req_ext -extfile /home/administrator/ca/csr/iota-csr.conf -batch