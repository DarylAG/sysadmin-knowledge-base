
# (BIND9 - Standalone & Master/Slave)

## Convenciones y Variables

Antes de aplicar esta configuración, reemplaza los siguientes marcadores de posición según tu entorno:
- `<DOMINIO>`: Nombre de tu dominio interno (ej. `dominio.com`).
- `<IP_MASTER>`: Dirección IP del servidor DNS principal.
- `<IP_SLAVE>`: Dirección IP del servidor DNS secundario (esclavo). Si configuras un servidor solo (standalone), ignora esta variable.
- `<SUBRED>`: Red local autorizada para hacer consultas (ej. `192.168.100.0/24`).
- `<ZONA_INVERSA>`: Octetos de red invertidos (ej. `100.168.192` para la red `192.168.100.0`).
- `<HOSTNAME_MASTER>`: Nombre del servidor principal (ej. `servidor01`).
- `<NOMBRE_LLAVE>`: Nombre de la llave TSIG para actualizaciones dinámicas (ej. `llave-actualizacion`).
## 1. Instalación de Paquetes
**Acción:** Instalar los paquetes necesarios para el servidor BIND9 y utilidades de DNS.
```
apt update && apt install -y bind9 bind9utils bind9-doc dnsutils
````

## 2. Configuración de Opciones Globales

**Acción:** Definir las redes autorizadas para realizar consultas y los reenviadores (forwarders) para resolver dominios externos.
```
nano /etc/bind/named.conf.options
```

```
# Establecemos una Lista de Control de Acceso (ACL) para definir quién puede consultar
acl "trusted" {
    127.0.0.1;
    <SUBRED>;  # Ej: 192.168.100.0/24
};

options {
    directory "/var/cache/bind";
    recursion yes;
    
    # Solo permitimos consultas recursivas a las IPs de nuestra ACL "trusted"
    allow-query { trusted; };
    
    # Reenviadores: Si este DNS no conoce la respuesta, preguntará a Google
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    dnssec-validation auto;
    
    # Escuchar en todas las interfaces IPv6 (opcional)
    listen-on-v6 { any; };
};
```

## 3. Declaración de Zonas Locales

**Acción:** Declarar las zonas DNS en el sistema. Si estás configurando un único servidor, o un servidor Maestro, usarás la configuración de "Master". Si estás en el segundo servidor, usarás la de "Slave".
```
nano /etc/bind/named.conf.local
```

```
# =====================================================================
# OPCIÓN A: CONFIGURACIÓN PARA SERVIDOR STANDALONE O MASTER
# =====================================================================

# Zona Directa (Traduce Nombres a IPs)
zone "<DOMINIO>" {
    type master;
    file "/etc/bind/zones/db.<DOMINIO>";
    
    # Si tienes un servidor Esclavo, descomenta las siguientes líneas:
    # allow-transfer { <IP_SLAVE>; };  # Permite al esclavo descargar la zona
    # also-notify { <IP_SLAVE>; };     # Notifica al esclavo cuando hay cambios
};

# Zona Inversa (Traduce IPs a Nombres)
zone "<ZONA_INVERSA>.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.<ZONA_INVERSA>";	
    
    # Si tienes un servidor Esclavo, descomenta las siguientes líneas:
    # allow-transfer { <IP_SLAVE>; }; 
    # also-notify { <IP_SLAVE>; }; 
};


# =====================================================================
# OPCIÓN B: CONFIGURACIÓN PARA SERVIDOR SLAVE (ESCLAVO)
# (Solo usar en el segundo servidor. No requiere crear archivos db.* manuales)
# =====================================================================

# zone "<DOMINIO>" {
#     type slave;
#     file "db.<DOMINIO>"; # Se guardará automáticamente en /var/cache/bind
#     masters { <IP_MASTER>; };
# };

# zone "<ZONA_INVERSA>.in-addr.arpa" {
#     type slave;
#     file "db.<ZONA_INVERSA>"; # Se guardará automáticamente en /var/cache/bind
#     masters { <IP_MASTER>; };
# };
```

## 4. Creación de Archivos de Zona (Solo Servidor Master)

_Nota: Estos archivos solo se crean en el servidor Maestro. El servidor Esclavo los descargará automáticamente._

**Acción:** Crear el directorio para almacenar las zonas locales.
```
mkdir -p /etc/bind/zones
```

**Acción:** Crear y editar el archivo de la zona directa (resuelve nombres a IPs).
```
nano /etc/bind/zones/db.<DOMINIO>
```

```
; Archivo de Zona Directa para <DOMINIO>
; Traduce nombres de host a direcciones IP

$TTL    604800
@       IN      SOA     <HOSTNAME_MASTER>.<DOMINIO>. admin.<DOMINIO>. (
                2026102401 ; Serial (YYYYMMDDNN) - Aumentar al hacer cambios
                604800     ; Refresh (1 semana)
                86400      ; Retry (1 día)
                2419200    ; Expire (4 semanas)
                604800     ; Negative cache TTL (1 semana)
)

; Servidores de Nombres (NS)
@       IN      NS      <HOSTNAME_MASTER>.<DOMINIO>.

; Registros tipo A (Hosts)
<HOSTNAME_MASTER>   IN      A       <IP_MASTER>
cliente01           IN      A       192.168.100.30
```

**Acción:** Crear y editar el archivo de la zona inversa (resuelve IPs a nombres).
```
nano /etc/bind/zones/db.<ZONA_INVERSA>
```

```
; Archivo de Zona Inversa para red <ZONA_INVERSA>
; Traduce direcciones IP a nombres de host

$TTL    604800
@       IN      SOA     <HOSTNAME_MASTER>.<DOMINIO>. admin.<DOMINIO>. (
                2026102401 ; Serial (YYYYMMDDNN) - Aumentar al hacer cambios
                604800     ; Refresh
                86400      ; Retry
                2419200    ; Expire
                604800     ; Negative cache TTL
)

; Servidores de Nombres (NS)
@       IN      NS      <HOSTNAME_MASTER>.<DOMINIO>.

; Registros tipo PTR (Punteros a nombres)
; El número de la izquierda corresponde al último octeto de la IP
1       IN      PTR     <HOSTNAME_MASTER>.<DOMINIO>.
30      IN      PTR     cliente01.<DOMINIO>.
```

## 5. Aplicar Permisos y Reiniciar

**Acción:** Asignar los permisos correctos al usuario `bind` y reiniciar el servicio para aplicar los cambios.
```
chown -R bind:bind /etc/bind/zones
systemctl restart bind9
systemctl enable bind9
```


## 6. Preparación para Actualizaciones Dinámicas (DDNS con DHCP)
*Nota: Si vas a integrar BIND9 con un servidor DHCP para que actualice los registros automáticamente, las zonas deben ubicarse en `/var/lib/bind/` para que el servicio tenga permisos de escritura.*

**Acción:** Crear el directorio para los archivos de zona dinámicos y copiar los archivos creados en el paso anterior.
```bash
mkdir -p /var/lib/bind/zones
cp /etc/bind/zones/db.* /var/lib/bind/zones/
````

**Acción:** Cambiar el propietario y los permisos del directorio para que BIND9 pueda escribir los cambios recibidos del DHCP.
```
chown -R bind:bind /var/lib/bind/zones
chmod -R 775 /var/lib/bind/zones
```

**Acción:** Generar la llave de seguridad criptográfica (TSIG) que permitirá al DHCP autenticarse y actualizar los registros DNS de forma segura.
```
tsig-keygen -a HMAC-SHA256 <NOMBRE_LLAVE> > /etc/bind/<NOMBRE_LLAVE>.key
chown root:bind /etc/bind/<NOMBRE_LLAVE>.key
chmod 640 /etc/bind/<NOMBRE_LLAVE>.key
```
_(Nota: Si cuentas con un servidor secundario/esclavo, debes enviar este archivo `.key` al otro servidor de forma segura, por ejemplo, usando `scp`)._

## 6. Gestión, Verificación y Troubleshooting

- **Verificar sintaxis de archivos globales:**
```
named-checkconf
```

- **Verificar sintaxis de zona directa e inversa (Solo en Master):**    
```
named-checkzone <DOMINIO> /etc/bind/zones/db.<DOMINIO>
named-checkzone <ZONA_INVERSA>.in-addr.arpa /etc/bind/zones/db.<ZONA_INVERSA>
```

- **Monitorear logs del servicio (útil para ver si la transferencia al esclavo funcionó):**
```
journalctl -u bind9 -f
```

- **Control del servicio:** `systemctl start/stop/restart/status bind9`

## Funciones Avanzadas y Próximos Pasos

- **DNS Dinámico (DDNS):** Integrar BIND9 con DHCP (ej. Kea DHCP) usando llaves TSIG para que los clientes actualicen sus registros DNS automáticamente al recibir una IP.

- **DNSSEC:** Firmar criptográficamente las zonas para garantizar que las respuestas DNS no sean alteradas (mitigación de DNS Spoofing).