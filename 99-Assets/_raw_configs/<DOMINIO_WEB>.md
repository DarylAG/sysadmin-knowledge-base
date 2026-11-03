```
# Redirección obligatoria de HTTP a HTTPS
server {
    # Puerto de escucha para HTTP
    listen 80;
    
    # Nombres de dominio que responderán a este bloque
    server_name <DOMINIO_WEB> www.<DOMINIO_WEB>;
    
    # Redirección 301 permanente hacia la versión segura
    return 301 https://$host$request_uri;
}

# Bloque principal para conexiones HTTPS
server {
    # Puerto de escucha para HTTPS
    listen 443 ssl;
    server_name <DOMINIO_WEB> www.<DOMINIO_WEB>;

    # Directorio raíz y archivos índice
    root <RUTA_RAIZ_WEB>/html;
    index index.html index.htm;

    # Rutas a los certificados SSL emitidos por la CA
    ssl_certificate /etc/ssl/certs/<CERTIFICADO_SSL>.crt;
    ssl_certificate_key /etc/ssl/private/<LLAVE_SSL>.key;

    # Protocolos y cifrados seguros permitidos
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Procesamiento de las peticiones a la raíz
    location / {
        try_files $uri $uri/ =404;
    }

    # Cabeceras de seguridad adicionales
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
}
```
