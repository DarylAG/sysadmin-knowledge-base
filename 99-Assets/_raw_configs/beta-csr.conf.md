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
commonName             = beta.xeon.cr

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = beta.xeon.cr
DNS.2 = mail.xeon.cr
DNS.3 = smtp.xeon.cr
DNS.4 = imap.xeon.cr
IP.1  = 192.168.1.2
```

