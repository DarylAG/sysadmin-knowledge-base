hostnamectl set-hostname iota

nano /etc/network/interfaces

nano /etc/hosts

nano /etc/resolv.conf

apt update && apt upgrade -y

apt update apt install squid -y

- `/etc/squid/certs/iota.crt`
    
- `/etc/squid/certs/iota.key`
    
- `/etc/squid/certs/delta-ca.crt`

- `/etc/squid/certs/delta-ca-sinpass.key`  ( cat delta-ca.crt delta-ca-sinpass.key > delta-ca.pem)

chown proxy:proxy /etc/squid/certs/delta-ca.pem 
chmod 600 /etc/squid/certs/delta-ca.pem

cd /etc/squid/certs/ 
cat delta-ca.crt delta-ca.key > delta-ca.pem chown proxy:proxy delta-ca.pem chmod 600 delta-ca.pem

chmod 755 /etc/squid/certs
chmod 600 /etc/squid/certs/iota.key
chown -R proxy:proxy /etc/squid/certs

mkdir -p /var/lib/squid
chown -R proxy:proxy /var/lib/squid

/usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB
chown -R proxy:proxy /var/lib/squid/ssl_db

mkdir -p /etc/squid/backup
cp /etc/squid/squid.conf /etc/squid./squid.conf.bak
mv /etc/squid/*.bak /etc/squid/backup/

nano /etc/squid/[[squid.conf.iota]]

squid -k parse

squid -z 
systemctl restart squid 
systemctl enable squid

# Redirección interna en el servidor iota (Linux)

apt update && apt install iptables -y

iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 3129
# En un cliente
curl -I -x http://192.168.106.4:3128 http://neverssl.com
