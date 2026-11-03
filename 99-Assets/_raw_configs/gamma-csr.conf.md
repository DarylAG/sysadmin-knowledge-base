```
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
prompt             = no

[ req_distinguished_name ]
countryName            = CR
stateOrProvinceName    = Alajuela
organizationName       = xeon.cr
commonName             = gamma.xeon.cr

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = gamma.xeon.cr
DNS.2 = www.xeon.cr
DNS.3 = xeon.cr
DNS.4 = gamma
IP.1  = 192.168.1.3
```

