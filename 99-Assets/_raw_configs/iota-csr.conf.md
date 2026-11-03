```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
C = CR
ST = Alajuela
L = Alajuela
O = xeon.cr
CN = iota.xeon.cr

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = iota.xeon.cr
DNS.2 = iota
IP.1 = 192.168.106.4
```

