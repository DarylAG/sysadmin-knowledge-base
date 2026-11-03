```
# Requerir SSL y definir rutas de certificados
ssl = required
ssl_server_cert_file = /etc/ssl/certs/<HOSTNAME_CORREO>.crt
ssl_server_key_file = /etc/ssl/private/<HOSTNAME_CORREO>.key
ssl_server_ca_file = /etc/ssl/certs/ca-root.crt

# Protocolo mínimo aceptado
ssl_min_protocol = TLSv1.2
```
