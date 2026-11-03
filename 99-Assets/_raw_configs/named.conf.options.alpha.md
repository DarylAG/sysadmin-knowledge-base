```bash
acl trusted {
        localhost;
        localnets;
        192.168.106.0/24;
};

options {
        directory "/var/cache/bind";
        recursion yes;
        allow-query { trusted; };
        allow-query-cache { trusted; };
        allow-recursion { trusted; };

		forwarders{
                8.8.8.8;
                8.8.4.4;
        };
        
        dnssec-validation auto;
        
        listen-on-v6 port 53 { ::1; };
        listen-on port 53 { 127.0.0.1; 192.168.106.1; };
};
```


