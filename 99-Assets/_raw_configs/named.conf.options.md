```
// Definir red confiable
acl trusted {
        localhost;
        localnets;
        <SUBRED_CIDR>;
};

options {
        directory "/var/cache/bind";
        recursion yes;
        
        // Permisos de consulta y recursión
        allow-query { trusted; };
        allow-query-cache { trusted; };
        allow-recursion { trusted; };

        // Servidores DNS externos
        forwarders{
                8.8.8.8;
                8.8.4.4;
        };
        
        dnssec-validation auto;
        listen-on-v6 port 53 { ::1; };
        listen-on port 53 { 127.0.0.1; <IP_SERVIDOR>; };
};
```
