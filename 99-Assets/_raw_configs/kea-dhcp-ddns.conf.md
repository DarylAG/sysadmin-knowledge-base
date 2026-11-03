
```JSON
{
"DhcpDdns":
{
  "ip-address": "127.0.0.1",
  "port": 53001,
  "control-socket": {
      "socket-type": "unix",
      "socket-name": "/var/run/kea/kea-ddns-ctrl-socket"
  },
  
  // Importar llave TSIG
  <?include "/etc/kea/tsig-keys.json"?>

  "forward-ddns" : {
      "ddns-domains" : [
          {
               "name": "<DOMINIO>.",
               "key-name": "<NOMBRE_LLAVE_TSIG>",
               "dns-servers": [ { "ip-address": "<IP_SERVIDOR>" } ]
          }
      ]
  },
  
  "reverse-ddns" : {
      "ddns-domains" : [
          {
               "name": "<ZONA_INVERSA_ARPA>.in-addr.arpa.",
               "key-name": "<NOMBRE_LLAVE_TSIG>",
               "dns-servers": [ { "ip-address": "<IP_SERVIDOR>" } ]
          }
      ]
  }
}
}
```