```
# Nivel de compatibilidad y dominios
compatibility_level = 3.9
mydomain = <DOMINIO_PRINCIPAL>
myorigin = $mydomain
myhostname = <FQDN_CORREO>
mydestination = $myhostname, <DOMINIO_PRINCIPAL>, localhost.$mydomain, localhost

# Red e Interfaces
inet_protocols = ipv4
inet_interfaces = all
mynetworks_style = host
mynetworks = 127.0.0.0/8 <RED_LOCAL>

# Buzones de correo (Maildir)
mailbox_size_limit = 0
home_mailbox = Maildir/
mailbox_command = 
recipient_delimiter = +
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases

# Configuración TLS
smtpd_tls_key_file = /etc/ssl/private/<HOSTNAME_CORREO>.key
smtpd_tls_cert_file = /etc/ssl/certs/<HOSTNAME_CORREO>.crt
smtp_tls_CAfile = /etc/ssl/certs/ca-root.crt
smtpd_tls_security_level = may
smtp_tls_security_level = may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_tls_auth_only = yes
smtpd_tls_loglevel = 1

# Autenticación SASL integrada con Dovecot
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname

# Restricciones de envío
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
```


