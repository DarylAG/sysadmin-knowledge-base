```bash
{
"DhcpDdns":
{
  "ip-address": "127.0.0.1",
  "port": 53001,
  "control-socket": {
      "socket-type": "unix",
      "socket-name": "/var/run/kea/kea-ddns-ctrl-socket"
  },
  
  <?include "/etc/kea/tsig-keys.json"?>

  "forward-ddns" : {
      "ddns-domains" : [
          {
               "name": "xeon.cr.",
               "key-name": "dhcp1-alpha",
               "dns-servers": [
                   { "ip-address": "192.168.106.1" }
               ]
          }
      ]
  },
  
  "reverse-ddns" : {
      "ddns-domains" : [
          {
               "name": "106.168.192.in-addr.arpa.",
               "key-name": "dhcp1-alpha",
               "dns-servers": [
                   { "ip-address": "192.168.106.1" }
               ]
          }
      ]
  },

  "loggers": [
    {
        "name": "kea-dhcp-ddns",
        "output_options": [
            {
                "output": "stdout",
                "pattern": "%-5p %m\n"
            }
        ],
        "severity": "INFO",
        "debuglevel": 0
    }
  ]
}
}
``` 
