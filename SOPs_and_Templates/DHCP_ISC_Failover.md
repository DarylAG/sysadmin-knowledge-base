# Implementación de Servidor DHCP (isc-dhcp-server)

## Convenciones y Variables
Antes de aplicar esta configuración, reemplaza los siguientes marcadores según tu entorno de red:
- `<INTERFAZ_RED>`: Interfaz por la cual el servidor entregará las IPs (ej. `enp0s3` o `eth0`).
- `<DOMINIO>`: Nombre de tu dominio interno (ej. `dominio.com`).
- `<IP_DNS_PRIMARIO>` / `<IP_DNS_SECUNDARIO>`: Direcciones IP de tus servidores DNS.
- `<SUBRED>`: Dirección de red local (ej. `192.168.100.0`).
- `<MASCARA>`: Máscara de subred (ej. `255.255.255.0`).
- `<RANGO_INICIO>` / `<RANGO_FIN>`: Rango de IPs a asignar a los clientes (ej. `192.168.100.110` y `192.168.100.120`).
- `<IP_NODO_PRIMARIO>`: IP de este servidor DHCP principal (Solo para Failover).
- `<IP_NODO_SECUNDARIO>`: IP del servidor DHCP de respaldo (Solo para Failover).

## 1. Instalación y Habilitación de Interfaces
**Acción:** Actualizar el sistema e instalar el paquete del servidor DHCP.
```
apt update && apt upgrade -y
apt install -y isc-dhcp-server
````

**Acción:** Indicarle al servicio por cuál interfaz de red debe escuchar las peticiones DHCP de los clientes.
```
nano /etc/default/isc-dhcp-server
```

```
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server) 
# Interfaces en las que el servidor DHCP debe escuchar peticiones. 
# Separa múltiples interfaces con espacios, ej. "eth0 eth1". 
INTERFACESv4="<INTERFAZ_RED>" 
INTERFACESv6=""
```

## 2. Configuración del Servicio (Elige una opción)
_Nota: Dependiendo de tu topología, edita el archivo principal usando la configuración Básica o la de Failover._

**Acción:** Configurar los parámetros globales, tiempos de concesión y el pool de direcciones IP.
```
nano /etc/dhcp/dhcpd.conf
```

- **Si es un servidor único (Standalone):** 
```
# Opciones Globales
option domain-name "<DOMINIO>";
option domain-name-servers <IP_DNS_PRIMARIO>, <IP_DNS_SECUNDARIO>;

# Declara que este es el servidor DHCP oficial para esta red
authoritative;

# Configuración de la Subred y el Pool de IPs
subnet <SUBRED> netmask <MASCARA> {
    pool {
        range <RANGO_INICIO> <RANGO_FIN>;
    }
    # Opciones específicas para esta subred (sobreescriben las globales si difieren)
    option domain-name "<DOMINIO>";
    option domain-name-servers <IP_DNS_PRIMARIO>;
    # option routers <IP_GATEWAY>; # Descomentar y agregar si hay salida a Internet
}
``` 

- **Si vas a usar Alta Disponibilidad (Failover):** 
```
# =====================================================================
# OPCIONES GLOBALES (Requerido en ambos nodos)
# =====================================================================
option domain-name "<DOMINIO>";
option domain-name-servers <IP_DNS_PRIMARIO>, <IP_DNS_SECUNDARIO>;
authoritative;


# =====================================================================
# PEER DE FAILOVER (Elige el bloque según el nodo donde estés)
# =====================================================================

# --- BLOQUE PARA NODO PRIMARIO ---
# failover peer "dhcp-failover" {
#     primary;
#     address <IP_NODO_PRIMARIO>;
#     port 647;
#     peer address <IP_NODO_SECUNDARIO>;
#     peer port 647;
#     max-response-delay 30;
#     max-unacked-updates 10;
#     load balance max seconds 3;
#     mclt 3600;
#     split 128;
}

# --- BLOQUE PARA NODO SECUNDARIO ---
# failover peer "dhcp-failover" {
#     secondary;
#     address <IP_NODO_SECUNDARIO>;
#     port 647;
#     peer address <IP_NODO_PRIMARIO>;
#     peer port 647;
#     max-response-delay 30;
#     max-unacked-updates 10;
#     load balance max seconds 3;
#     mclt 3600;
#     split 128;
# }


# =====================================================================
# SUBRED Y POOL CON ALTA DISPONIBILIDAD (Igual en ambos nodos)
# =====================================================================
subnet <SUBRED> netmask <MASCARA> {
    pool {
        failover peer "dhcp-failover";
        range <RANGO_INICIO> <RANGO_FIN>;
    }
    option domain-name "<DOMINIO>";
    option domain-name-servers <IP_DNS_PRIMARIO>;
}
```
_(asegúrate de usar el bloque correcto según si estás en el nodo Primario o Secundario)_.

## 3. Aplicar Cambios
**Acción:** Reiniciar el servicio para cargar la nueva configuración y habilitarlo para que inicie con el sistema.
```
systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
```

## 4. Gestión, Verificación y Troubleshooting

- **Verificar sintaxis del archivo de configuración (CRÍTICO antes de reiniciar):**
```
dhcpd -t
```

- **Monitorear logs en tiempo real (útil para ver los DORA: Discover, Offer, Request, Acknowledge):**    
```
journalctl -u isc-dhcp-server -f
```
 
- **Control del servicio:** `systemctl start/stop/restart/status isc-dhcp-server`

- **Ver concesiones activas (Leases):**
```
cat /var/lib/dhcp/dhcpd.leases
```

## Funciones Avanzadas y Próximos Pasos

- **Reservas de IP (Static Leases):** Asignar direcciones IP fijas a dispositivos específicos basándose en su dirección MAC.

- **Integración DDNS:** Habilitar la actualización automática de registros en el servidor DNS (BIND9) cuando un cliente recibe una IP.