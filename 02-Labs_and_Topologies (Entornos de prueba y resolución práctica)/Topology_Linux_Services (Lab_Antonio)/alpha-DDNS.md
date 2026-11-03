```bash
hostnamectl set-hostname alpha
nano /etc/network/interfaces
	allow-hotplug enp0s3
	iface enp0s3 inet static
		address 192.168.106.1
		netmask 255.255.255.0
		gateway 192.168.106.254
		
nano /etc/resolv.conf
	nameserver 8.8.8.8
	search xeon.cr
	
apt update && apt upgrade -y

```
## kea-dhcp
```bash
apt install curl apt-transport-https -y
curl -1sLf \ 'https://dl.cloudsmith.io/public/isc/kea-3-2/setup.deb.sh' \ | bash
apt install isc-kea-dhcp4-server -y
cd /etc/kea
mv kea-dhcp4.conf kea-dhcp4.conf.bak
```
nano [[kea-dhcp4.conf.alpha]]
```bash 
chown _kea kea-dhcp4.conf
chmod 640 kea-dhcp4.conf
systemctl restart isc-kea-dhcp4-server.service
systemctl enable isc-kea-dhcp4-server.service
```
## bind9
```bash
apt install -y bind9 bins9utils bind9-doc dnsutils
cd /etc/bind
cp named.conf.options named.conf.options.bak
```
nano  [[named.conf.options.alpha]]
```bash
cp named.conf.local named.conf.local.bak
```
nano [[named.conf.local.alpha]]
```bash
mkdir zones
nano ./zones/db.xeon.cr ------> 
nano ./zones/db.192.168.106 ------>
nano /etc/resolv.conf 
	nameserver 127.0.0.1
	search xeon.cr
```
## DDNS
#bind
```bash
cd /etc/bind
tsig-keygen dhcp1-alpha > dhcp1-alpha.key
chmod 640 dhcp1-alpha.key
mv ./zones/db.* /var/lib/bind/
cp named.conf.local named.conf.local.old
```
nano [[named.conf.local.alpha]]
```bash
systemctl restart bind9
systemctl status bind9
```
#kea-dhcp
```bash
apt install isc-kea-dhcp-ddns-server -y
cd /etc/kea
awk -F'"' '/secret/ {printf "%s", $2}' /etc/bind/dhcp1-alpha.key > /etc/kea/dhcp1-alpha.secret
nano tsig-keys.json
	"tsig-keys": [
	    {
	       "name": "dhcp1-alpha",
	       "algorithm": "hmac-sha256",
	       "secret-file": "/etc/kea/dhcp1-alpha.secret"
	    }
	],


chown _kea:_kea /etc/kea/tsig-keys.json
chmod 640 /etc/kea/tsig-keys.json
cp kea-dhcp-ddns.conf kea-dhcp-ddns.conf.bak
```
nano [[kea-dhcp-ddns.conf.alpha]]
```bash
chown _kea:_kea /etc/kea/kea-dhcp-ddns.conf
chmod 640 /etc/kea/kea-dhcp-ddns.conf
kea-dhcp-ddns -t kea-dhcp-ddns.conf
systemctl restart isc-kea-dhcp-ddns-server
systemctl status isc-kea-dhcp-ddns-server
```
nano [[kea-dhcp4.conf.alpha]]
```bash
	 "dhcp-ddns": {
        "enable-updates": true
     },
     
     "ddns-qualifying-suffix": "xeon.cr",
     "ddns-override-client-update": true,

kea-dhcp4 -t kea-dhcp4.conf
systemctl restart isc-kea-dhcp4-server
systemctl status isc-kea-dhcp4-server
```




# Service Status and Logs
### kea-dhcp
```bash
journalctl -u isc-kea-dhcp4-server.service -e --no-pager
systemctl status isc-kea-dhcp4-server.service
more /var/lib/kea/leases4.csv

systemctl start isc-kea-dhcp-ddns-server 
systemctl status isc-kea-dhcp-ddns-server
```
### Bind9
```bash
named-checkzone xeon.cr db.xeon.cr
named-checkzone 106.168.192.in-addr.arpa db.192.168.106
```
## Maintenance
**Pause Updates to make changes in the zones**
`rndc freeze`
**Unpaused**
`rndc thaw` 