```
<IfModule mod_tls.c>
    # Habilitar el motor criptográfico TLS
    TLSEngine on

    # Ruta del archivo de registro específico para eventos y errores TLS
    TLSLog /var/log/proftpd/tls.log

    # Restringir las conexiones a protocolos seguros modernos
    TLSProtocol TLSv1.2 TLSv1.3

    # Rutas del certificado público y la llave privada del servidor
    TLSRSACertificateFile /etc/ssl/certs/<CERTIFICADO_SSL>.crt
    TLSRSACertificateKeyFile /etc/ssl/private/<LLAVE_SSL>.key

    # Ruta del certificado de la Autoridad Certificadora (CA) de confianza
    TLSCACertificateFile /etc/ssl/certs/<CERTIFICADO_CA>.crt

    # Desactivar la exigencia de un certificado del lado del cliente
    TLSVerifyClient off

    # FORZAR a los clientes a utilizar FTP sobre TLS (rechaza conexiones de texto plano)
    TLSRequired on
</IfModule>
```