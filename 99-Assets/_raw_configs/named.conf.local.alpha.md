# DNS 
```bash
zone "xeon.cr" {
	type master;
	file "/etc/bind/zones/db.xeon.cr";
};

zone "106.168.192.in-addr.arpa" {
	type master;
	file "/etc/bind/zones/db.192.168.106";	

};
```
# DDNS

```bash 
include "/etc/bind/dhcp1-alpha.key";

zone "xeon.cr" {
    type master;
    file "/var/lib/bind/db.xeon.cr";
    update-policy {
        grant dhcp1-alpha wildcard *.xeon.cr A DHCID;
    };
};

zone "106.168.192.in-addr.arpa" {
	type master;
	file "/var/lib/bind/db.192.168.106";    
    update-policy {
        grant dhcp1-alpha wildcard *.106.168.192.in-addr.arpa PTR DHCID;
   };
};
``` 

