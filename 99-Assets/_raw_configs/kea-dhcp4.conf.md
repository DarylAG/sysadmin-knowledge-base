```JSON
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": ["<INTERFAZ_RED>"]
    },

    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv",
        "lfc-interval": 3600
    },

    // Temporizadores de concesión
    "renew-timer": 15840,
    "rebind-timer": 27720,
    "valid-lifetime": 31680,

    // Habilitar actualizaciones DDNS dinámicas
    "dhcp-ddns": {
        "enable-updates": true
    },
    "ddns-qualifying-suffix": "<DOMINIO>",
    "ddns-override-client-update": true,

    "subnet4": [
        {
            "id": 1,
            "subnet": "<SUBRED_CIDR>",
            "pools": [ { "pool": "<RANGO_DHCP>" } ],
            "option-data": [
                { "name": "routers", "data": "<IP_GATEWAY>" },
                { "name": "domain-name-servers", "data": "<IP_SERVIDOR>" },
                { "name": "domain-search", "data": "<DOMINIO>" }
            ]
        }
    ]
}
}
```

