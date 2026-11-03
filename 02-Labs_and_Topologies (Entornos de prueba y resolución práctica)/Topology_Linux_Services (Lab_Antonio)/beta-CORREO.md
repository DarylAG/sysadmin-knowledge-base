hostnamectl set-hostname beta
nano /etc/hosts
	127.0.0.1       localhost
	127.0.1.1       beta.xeon.cr    beta

nano /etc/resolv.conf
	search xeon.cr
	nameserver 192.168.106.1

nano /etc/network/interfaces
	# The primary network interface
	allow-hotplug enp0s3
	iface enp0s3 inet static
        address 192.168.106.2
        netmask 255.255.255.0
        dns-search xeon.cr
        dns-nameserver 192.168.106.1
        gateway 192.168.106.254

apt update && apt upgrade -y

scp administrator@delta:/home/administrator/ca/certs/beta.crt /etc/ssl/certs/beta.crt
scp administrator@delta:/home/administrator/ca/private/beta.key /etc/ssl/private/beta.key
scp administrator@delta:/home/administrator/ca/certs/delta-ca.crt /etc/ssl/certs/delta-ca.crt

chmod 600 /etc/ssl/private/beta.key
chown root:root /etc/ssl/private/beta.key

apt update && apt install -y postfix mailutils
cp /etc/postfix/[[main.cf.beta|main.cf]] /etc/postfix/main.cf.bak
nano [[main.cf.beta|main.cf]] 
cp /etc/postfix/master.cf /etc/postfix/master.cf.bak
nano [[main.cf.beta|main.cf]]

#Dovecot
apt install -y dovecot-imapd dovecot-pop3d
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.bak
nano /etc/dovecot/conf.d/[[10-ssl.conf.beta]]

cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.bak
nano /etc/dovecot/conf.d/[[10-auth.conf.beta]]

cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.bak
nano /etc/dovecot/conf.d/[[10-master.conf.beta]]

systemctl restart postfix
systemctl restart dovecot

systemctl status postfix
systemctl status dovecot
openssl s_client -connect beta.xeon.cr:993 -CAfile /etc/ssl/certs/delta-ca.crt

useradd -m -s /sbin/nologin -c "Usuario de Correo XEON" correo1
passwd correo1

mkdir -p /home/correo1/Maildir/{new,cur,tmp}
chown -R correo1:correo1 /home/correo1/Maildir/
chmod -R 700 /home/correo1/Maildir/
mkdir -p /home/correo2/Maildir/{new,cur,tmp}
chown -R correo2:correo2 /home/correo2/Maildir/
chmod -R 700 /home/correo2/Maildir/
