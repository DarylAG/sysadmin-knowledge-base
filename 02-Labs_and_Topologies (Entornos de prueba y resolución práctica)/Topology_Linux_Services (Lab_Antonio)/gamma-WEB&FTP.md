
hostnamectl set-hostname gamma

nano /etc/hosts
	127.0.0.1       localhost
	127.0.1.1       gamma.xeon.cr   gamma

nano /etc/network/interfaces
	# The primary network interface
	allow-hotplug enp0s3
	iface enp0s3 inet static
        address 192.168.106.3
        netmask 255.255.255.0
        gateway 192.168.106.254

nano /etc/resolv.conf
	nameserver 192.168.106.1
	search xeon.cr

apt update && upt upgrade -y

#nginx
apt install nginx -y

mkdir -p /var/www/xeon.cr/html

nano /etc/nginx/sites-available/xeon.cr

scp administrator@delta:~/ca/private/gamma.key ./
mv gamma.key /etc/ssl/private/
chown root:root /etc/ssl/private/gamma.key
chmod 600 /etc/ssl/private/gamma.key

scp administrator@delta:~/ca/private/delta-ca.crt ./
mv delta-ca.crt /etc/ssl/certs/
chown root:root /etc/ssl/private/delta-ca.crt
chmod 644 /etc/ssl/private/delta-ca.crt


scp administrator@delta:~/ca/private/gamma.crt ./
mv gamma.crt /etc/ssl/certs/
chown root:root /etc/ssl/private/gamma.crt
chmod 644 /etc/ssl/private/gamma.crt


ln -s /etc/nginx/sites-available/xeon.cr /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx

#webamin
useradd -m -d /var/www/xeon.cr -s /usr/sbin/nologin webadmin 
passwd webadmin

chown -R webadmin:www-data /var/www/xeon.cr
chmod -R 775 /var/www/xeon.cr

#ProFTPD

apt install proftpd-basic -y
apt install proftpd-mod-crypto -y

nano /etc/proftpd/proftpd.conf

nano /etc/proftpd/tls.conf

nano /etc/proftpd/modules.conf



