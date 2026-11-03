```
// Incluir la llave generada
include "/etc/bind/<NOMBRE_LLAVE_TSIG>.key";

// Zona Directa
zone "<DOMINIO>" {
    type master;
    // Las zonas deben estar en /var/lib/bind para que el usuario bind tenga permisos de escritura en DDNS
    file "/var/lib/bind/db.<DOMINIO>";
    update-policy {
        grant <NOMBRE_LLAVE_TSIG> wildcard *.<DOMINIO> A DHCID;
    };
};

// Zona Inversa
zone "<ZONA_INVERSA_ARPA>.in-addr.arpa" {
    type master;
    file "/var/lib/bind/db.<ZONA_INVERSA_ARPA>";    
    update-policy {
        grant <NOMBRE_LLAVE_TSIG> wildcard *.<ZONA_INVERSA_ARPA>.in-addr.arpa PTR DHCID;
   };
};
```
