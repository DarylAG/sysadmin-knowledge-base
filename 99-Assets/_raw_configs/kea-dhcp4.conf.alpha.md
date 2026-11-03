```bash
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": ["enp0s3"]
    },

    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv",
        "lfc-interval": 3600
    },

    "renew-timer": 15840,
    "rebind-timer": 27720,
    "valid-lifetime": 31680,


    "subnet4": [
        {
	    "id": 1,
            "subnet": "192.168.106.0/24",
            "pools": [ { "pool": "192.168.106.100 - 192.168.106.199" } ],
            "option-data": [
                {
                    "name": "routers",
                    "data": "192.168.106.254"
                },
				{
				    "name": "domain-name-servers",
				    "data": "192.168.106.1"
				},
				{
				    "name": "domain-search",
				    "data": "xeon.cr"
				}
            ]
            
            // Add reservations here
        }
        
        // Add subnets here
    ]
}
}
```

# DDNS
```bash
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": ["enp0s3"]
    },

    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv",
        "lfc-interval": 3600
    },

    "renew-timer": 15840,
    "rebind-timer": 27720,
    "valid-lifetime": 31680,

    "dhcp-ddns": {
	"enable-updates": true
    },

    "ddns-qualifying-suffix": "xeon.cr",
    "ddns-override-client-update": true,

    "subnet4": [
        {
	    "id": 1,
            "subnet": "192.168.106.0/24",
            "pools": [ { "pool": "192.168.106.100 - 192.168.106.199" } ],
            "option-data": [
                {
                    "name": "routers",
                    "data": "192.168.106.254"
                },
		{
		    "name": "domain-name-servers",
		    "data": "192.168.106.1"
		},
		{
		    "name": "domain-search",
		    "data": "xeon.cr"
		}
            ]
            
            // Add reservations here
        }
        
        // Add subnets here
    ]
}
}

```
