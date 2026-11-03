# Convenciones y Variables

Esta guía utiliza marcadores de posición genéricos. Antes de ejecutar los comandos o copiar las configuraciones, reemplaza los siguientes valores según tu entorno de red:

- `<HOSTNAME_PRIMARIO>`: Nombre del servidor (ej. `alpha`).

- `<DOMINIO>`: Nombre de tu dominio interno (ej. `xeon.cr`).

- `<INTERFAZ_RED>`: Interfaz de red principal (ej. `enp0s3`).

- `<IP_SERVIDOR>`: Dirección IP estática del servidor (ej. `192.168.106.1`).

- `<IP_GATEWAY>`: Dirección IP de la puerta de enlace (ej. `192.168.106.254`).

- `<SUBRED_CIDR>`: Subred en formato CIDR (ej. `192.168.106.0/24`).

- `<ZONA_INVERSA_ARPA>`: Octetos de red invertidos para la zona PTR (ej. `106.168.192`).

- `<RANGO_DHCP>`: Rango de IPs a asignar por DHCP (ej. `192.168.106.100 - 192.168.106.199`).

- `<NOMBRE_LLAVE_TSIG>`: Nombre de la llave para actualizaciones DDNS (ej. `dhcp1-alpha`).

## 1. Preparación del Sistema y Red

Configurar el nombre de host del servidor.
```
hostnamectl set-hostname <HOSTNAME_PRIMARIO>
```

Configurar la interfaz de red con IP estática.
```
nano /etc/network/interfaces
```

```
allow-hotplug <INTERFAZ_RED>
iface <INTERFAZ_RED> inet static
    address <IP_SERVIDOR>
    netmask 255.255.255.0
    gateway <IP_GATEWAY>
```

Actualizar el sistema base.
```
apt update && apt upgrade -y
```

# 2. Instalación y Configuración de BIND9 (DNS)

Instalar los paquetes requeridos para el servidor DNS.
```
apt install -y bind9 bind9utils bind9-doc dnsutils
```

Generar la llave TSIG para asegurar las actualizaciones dinámicas (DDNS).
```
cd /etc/bind
tsig-keygen <NOMBRE_LLAVE_TSIG> > <NOMBRE_LLAVE_TSIG>.key
chmod 640 <NOMBRE_LLAVE_TSIG>.key
```

Configurar las opciones globales de BIND9.
```
nano /etc/bind/named.conf.options
```


# 2. Instalación y Configuración de BIND9 (DNS)

Instalar los paquetes requeridos para el servidor DNS.
```
apt install -y bind9 bind9utils bind9-doc dnsutils
```

Generar la llave TSIG para asegurar las actualizaciones dinámicas (DDNS).
```
cd /etc/bind
tsig-keygen <NOMBRE_LLAVE_TSIG> > <NOMBRE_LLAVE_TSIG>.key
chmod 640 <NOMBRE_LLAVE_TSIG>.key
```

Configurar las opciones globales de BIND9.
```
nano /etc/bind/named.conf.options
```
[[named.conf.options|named.conf.options]]

Configurar las zonas locales permitiendo la actualización mediante la llave TSIG.
```
nano /etc/bind/named.conf.local
```
[[named.conf.local|named.conf.local]]

Crear el archivo de la zona directa.
```
nano /var/lib/bind/db.<DOMINIO>
```
[[db.<DOMINIO>]]

Crear el archivo de la zona inversa.
```
nano /var/lib/bind/db.<ZONA_INVERSA_ARPA>
```
[[db.<ZONA_INVERSA_ARPA>]]

Corregir permisos y reiniciar BIND9.
```
chown bind:bind /var/lib/bind/db.*
systemctl restart bind9
```


## 3. Instalación y Configuración de Kea DHCP y DDNS

Instalar el repositorio de Kea y los paquetes del servidor DHCP y DDNS.
```
apt install curl apt-transport-https -y
curl -1sLf 'https://dl.cloudsmith.io/public/isc/kea-3-2/setup.deb.sh' | bash
apt install isc-kea-dhcp4-server isc-kea-dhcp-ddns-server -y
```

Extraer el secreto TSIG de BIND9 para usarlo en Kea.
```
cd /etc/kea
awk -F'"' '/secret/ {printf "%s", $2}' /etc/bind/<NOMBRE_LLAVE_TSIG>.key > /etc/kea/<NOMBRE_LLAVE_TSIG>.secret
```

Configurar el archivo de llaves TSIG para Kea.
```
nano /etc/kea/tsig-keys.json
```

```JSON
{
    "tsig-keys": [
        {
           "name": "<NOMBRE_LLAVE_TSIG>",
           "algorithm": "hmac-sha256",
           "secret-file": "/etc/kea/<NOMBRE_LLAVE_TSIG>.secret"
        }
    ]
}
```

Aplicar permisos a las llaves.
```
chown _kea:_kea /etc/kea/tsig-keys.json
chmod 640 /etc/kea/tsig-keys.json
```

Configurar el módulo Kea DDNS (D2).
```
nano /etc/kea/kea-dhcp-ddns.conf
```

Configurar el servidor DHCPv4 con integración DDNS.
```
nano /etc/kea/kea-dhcp4.conf
```

Reiniciar y habilitar los servicios de Kea.
```
systemctl restart isc-kea-dhcp-ddns-server
systemctl restart isc-kea-dhcp4-server
systemctl enable isc-kea-dhcp-ddns-server isc-kea-dhcp4-server
```


# 4. Gestión, Verificación y Troubleshooting

## Comandos de Control de Servicios
```
systemctl start/stop/restart/status bind9
systemctl start/stop/restart/status isc-kea-dhcp4-server
systemctl start/stop/restart/status isc-kea-dhcp-ddns-server
```

## Comandos de Verificación de Sintaxis

Para validar la configuración de DNS (BIND9):
```
named-checkconf
named-checkzone <DOMINIO> /var/lib/bind/db.<DOMINIO>
named-checkzone <ZONA_INVERSA_ARPA>.in-addr.arpa /var/lib/bind/db.<ZONA_INVERSA_ARPA>
```

Para validar los archivos de configuración JSON de Kea:
```
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
kea-dhcp-ddns -t /etc/kea/kea-dhcp-ddns.conf
```

## Mantenimiento de Zonas DDNS (BIND9)

Antes de modificar manualmente un archivo de zona que está siendo actualizado dinámicamente, pausa las actualizaciones:
```
rndc freeze <DOMINIO>
```

Una vez realizados los cambios, reanuda las actualizaciones:
```
rndc thaw <DOMINIO>
```

### Troubleshooting y Logs

Visualizar los logs del servicio DHCP en tiempo real:
```
journalctl -u isc-kea-dhcp4-server.service -e --no-pager
```

Verificar la base de datos de concesiones (leases) en Kea:
```
more /var/lib/kea/kea-leases4.csv
```

# 5. Funciones Avanzadas y Próximos Pasos

- **Implementación de DNSSEC**: Firmar criptográficamente las zonas para garantizar la integridad y autenticidad de las respuestas DNS.
    
- **Alta Disponibilidad (HA)**: Configurar un servidor BIND9 secundario (esclavo) con transferencias de zona (`AXFR`) y establecer el esquema de alta disponibilidad nativo de Kea DHCP para redundancia activa-pasiva o activa-activa.
    
- **Base de datos backend para Kea**: Migrar el almacenamiento de concesiones (leases) del archivo CSV en memoria a un motor relacional sólido (como PostgreSQL o MySQL) para entornos empresariales escalables.