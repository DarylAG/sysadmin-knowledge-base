```
$TTL 604800	; 1 week
<ZONA_INVERSA_ARPA>.in-addr.arpa. IN SOA <HOSTNAME_PRIMARIO>.<DOMINIO>. admin.<DOMINIO>. (
				2026101731 ; serial
				604800     ; refresh 
				86400      ; retry 
				2419200    ; expire 
				604800     ; minimum 
				)
			NS	<HOSTNAME_PRIMARIO>.<DOMINIO>.
1.<ZONA_INVERSA_ARPA>.in-addr.arpa. PTR	<HOSTNAME_PRIMARIO>.<DOMINIO>.
```