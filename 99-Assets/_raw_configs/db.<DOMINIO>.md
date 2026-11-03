```
$TTL 604800	; 1 week
<DOMINIO>.		IN SOA	<HOSTNAME_PRIMARIO>.<DOMINIO>. admin.<DOMINIO>. (
				2026102249 ; serial
				604800     ; refresh 
				86400      ; retry 
				2419200    ; expire 
				604800     ; minimum 
				)
			NS	<HOSTNAME_PRIMARIO>.<DOMINIO>.
<HOSTNAME_PRIMARIO>.<DOMINIO>.		A	<IP_SERVIDOR>
```
