```bash
$TTL 604800	; 1 week
xeon.cr.		IN SOA	alpha.xeon.cr. admin.xeon.cr. (
				2026102249 ; serial
				604800     ; refresh (1 week)
				86400      ; retry (1 day)
				2419200    ; expire (4 weeks)
				604800     ; minimum (1 week)
				)
			NS	alpha.xeon.cr.
			MX	10 beta.xeon.cr.
alpha.xeon.cr.		A	192.168.106.1
beta.xeon.cr.		A	192.168.106.2
imap.xeon.cr.		A	192.168.106.2
smtp.xeon.cr.		A	192.168.106.2
mail.xeon.cr.		A	192.168.106.2
gamma.xeon.cr.		A	192.168.106.3
www.xeon.cr.		A	192.168.106.3
iota.xeon.cr.		A	192.168.106.4




```