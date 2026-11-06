# Standard Operating Procedure (SOP) - Mail Server (Postfix & Dovecot)

## Overview and Laboratory Topology

This Standard Operating Procedure (SOP) outlines the deployment of an enterprise-grade **Mail Transfer Agent (MTA) and Mail Delivery Agent (MDA)** using **Postfix** and **Dovecot** within a simulated multi-node network infrastructure in GNS3. The mail server provides secure internal communication (SMTP Submission over TLS, IMAPS/POP3S) utilizing certificates issued by the internal Root CA.

In this environment, all servers and client hosts are interconnected through a central switch (`Switch1`) and routed to the external network/Internet via router `R1` and a `NAT1` cloud interface:

- **Topology Naming Scheme & Static IPs:**
    
    - `alpha-DDNS-1`: DNS & DHCP Server (`.1`)
    
    - **`beta-CORREO-1`**: Mail Server (Postfix/Dovecot) (`.2`) - _Primary target of this SOP_
    
    - `gamma-WEB&FTP-1`: Web & FTP Server (`.3`)
    
    - `iota-PROXY-1`: Squid Proxy Server (`.4`)
    
    - `delta-CA-1`: Internal Certificate Authority (CA) (`.5`)
    
    - `pcw01-1` / `pcl01-1`: Windows and Linux Client Workstations
    

<p align="center">
  <img src="../Assets/GNS3_LAB" alt="GNS3 Lab Topology" width="80%">
</p>

Since the mail server is deployed directly into this pre-addressed topology, initial network resolution and static IP configuration are verified prior to installing mail suites and enforcing TLS authentication.

---

## Conventions and Environment Variables

To ensure the adaptability of this template to any environment, static values have been replaced with placeholders. Before executing the commands, replace the following variables with the actual values for your infrastructure:

**Environment and Naming Variables**

- `<DOMAIN_NAME>`: Primary email domain name (e.g., `xeon.cr`).

- `<MAIL_HOSTNAME>`: Short hostname of the mail server (e.g., `beta`).

- `<MAIL_FQDN>`: Fully Qualified Domain Name of the mail server (e.g., `beta.xeon.cr`).

- `<SERVER_IP>`: Static IP address assigned to this server (e.g., `192.168.106.2`).

- `<GATEWAY_IP>`: Default gateway IP address (e.g., `192.168.106.254`).

- `<DNS_IP>`: Internal DNS resolver IP address (e.g., `192.168.106.1`).

- `<LOCAL_NET>`: Subnet allowed for local mail relaying (e.g., `192.168.106.0/24`).

- `<MAIL_USER>`: System user account created for mailbox access (e.g., `jdoe`).

- `<CA_USER>@<CA_HOST>`: SSH user and IP of the Certificate Authority host (e.g., `root@192.168.106.5`).

---

## 1. System and Network Preparation

Set the short hostname for the mail server.
```bash
hostnamectl set-hostname <MAIL_HOSTNAME>
```

Configure local host resolution to map the loopback interface to the server's FQDN and short hostname.
```bash
nano /etc/hosts
```

```conf
127.0.0.1       localhost
127.0.1.1       <MAIL_FQDN>    <MAIL_HOSTNAME>
```

Configure the system DNS resolver pointing to the internal DNS server.
```bash
nano /etc/resolv.conf
```

```conf
search <DOMAIN_NAME>
nameserver <DNS_IP>
```

Assign the static IP parameters on the primary network interface (adjust `enp0s3` according to your interface name).
```bash
nano /etc/network/interfaces
```

```conf
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
    address <SERVER_IP>
    netmask 255.255.255.0
    gateway <GATEWAY_IP>
```

Update system package index and upgrade installed packages.
```bash
apt update && apt upgrade -y
```

---

## 2. SSL/TLS Certificate Deployment

Securely copy the public certificate, private key, and Root CA certificate issued by `CA-Authority`.
```bash
scp <CA_USER>@<CA_HOST>:/root/ca/certs/<MAIL_HOSTNAME>.crt /etc/ssl/certs/<MAIL_HOSTNAME>.crt
scp <CA_USER>@<CA_HOST>:/root/ca/private/<MAIL_HOSTNAME>.key /etc/ssl/private/<MAIL_HOSTNAME>.key
scp <CA_USER>@<CA_HOST>:/root/ca/certs/ca-root.crt /etc/ssl/certs/ca-root.crt
```

Enforce strict access permissions on the private key file.
```bash
chmod 600 /etc/ssl/private/<MAIL_HOSTNAME>.key
chown root:root /etc/ssl/private/<MAIL_HOSTNAME>.key
```

## 3. Postfix (MTA) Installation and Configuration

Install the Postfix Mail Transfer Agent and mail utility binaries.
```
apt update && apt install -y postfix mailutils
```

Backup and configure the main Postfix configuration file (`/etc/postfix/main.cf`).
```bash 
cp /etc/postfix/main.cf /etc/postfix/main.cf.bak
nano /etc/postfix/main.cf
```

```conf
# Compatibility level and domain definitions
compatibility_level = 3.9
mydomain = <DOMAIN_NAME>
myorigin = $mydomain
myhostname = <MAIL_FQDN>
mydestination = $myhostname, <DOMAIN_NAME>, localhost.$mydomain, localhost

# Network and Interfaces
inet_protocols = ipv4
inet_interfaces = all
mynetworks_style = host
mynetworks = 127.0.0.0/8 <LOCAL_NET>

# Mailbox management (Maildir format)
mailbox_size_limit = 0
home_mailbox = Maildir/
mailbox_command = 
recipient_delimiter = +
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases

# TLS Security Parameters
smtpd_tls_key_file = /etc/ssl/private/<MAIL_HOSTNAME>.key
smtpd_tls_cert_file = /etc/ssl/certs/<MAIL_HOSTNAME>.crt
smtp_tls_CAfile = /etc/ssl/certs/ca-root.crt
smtpd_tls_security_level = may
smtp_tls_security_level = may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtpd_tls_auth_only = yes
smtpd_tls_loglevel = 1

# SASL Authentication via Dovecot
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname

# Recipient and Relay Restrictions
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
```

Backup and configure the Postfix master process file (`/etc/postfix/master.cf`) to enable secure SMTP Submission (Port 587).
```bash
cp /etc/postfix/master.cf /etc/postfix/master.cf.bak
nano /etc/postfix/master.cf
```

```conf
smtp      inet  n       -       y       -       -       smtpd



submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
pickup    unix  n       -       y       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       y       1000?   1       tlsmgr
rewrite   unix  -       -       y       -       -       trivial-rewrite
bounce    unix  -       -       y       -       0       bounce
defer     unix  -       -       y       -       0       bounce
trace     unix  -       -       y       -       0       bounce
verify    unix  -       -       y       -       1       verify
flush     unix  n       -       y       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       y       -       -       smtp
relay     unix  -       -       y       -       -       smtp
        -o syslog_name=${multi_instance_name?{$multi_instance_name}:{postfix}}/$service_name
showq     unix  n       -       y       -       -       showq
error     unix  -       -       y       -       -       error
retry     unix  -       -       y       -       -       error
discard   unix  -       -       y       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       y       -       -       lmtp
anvil     unix  -       -       y       -       1       anvil
scache    unix  -       -       y       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd

```

---

## 4. Dovecot (IMAP/POP3) Installation and Configuration

Install Dovecot IMAP and POP3 service packages.
```bash
apt install -y dovecot-imapd dovecot-pop3d
```

Configure SSL parameters (`/etc/dovecot/conf.d/10-ssl.conf`) to enforce TLS encryption.
```bash
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.bak
nano /etc/dovecot/conf.d/10-ssl.conf
```

```conf
# Enforce SSL/TLS and specify certificate paths
ssl = required
ssl_server_cert_file = /etc/ssl/certs/<MAIL_HOSTNAME>.crt
ssl_server_key_file = /etc/ssl/private/<MAIL_HOSTNAME>.key
ssl_server_ca_file = /etc/ssl/certs/ca-root.crt

# Minimum TLS protocol version
ssl_min_protocol = TLSv1.2
```

Configure system authentication mechanisms (`/etc/dovecot/conf.d/10-auth.conf`).
```bash
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.bak
nano /etc/dovecot/conf.d/10-auth.conf
```

```conf
!include auth-system.conf.ext

```

Configure Dovecot service sockets (`/etc/dovecot/conf.d/10-master.conf`) to handle authentication requests for Postfix.
```bash
cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.bak
nano /etc/dovecot/conf.d/10-master.conf
```

```conf
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}

service auth {
  unix_listener auth-userdb {
    mode = 0660
    user = postfix
    group = postfix
  }

  # Postfix SASL authentication socket
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}

service auth-worker {
  user = root
}

```

---

## 5. User Creation and Maildir Provisioning

Create system users without shell access (`nologin`) to serve as dedicated email accounts.
```bash
useradd -m -s /sbin/nologin -c "Mail User Account" <MAIL_USER>
passwd <MAIL_USER>
```

Initialize the `Maildir` directory tree (`new`, `cur`, `tmp`) with strict owner-only permissions.
```bash
mkdir -p /home/<MAIL_USER>/Maildir/{new,cur,tmp}
chown -R <MAIL_USER>:<MAIL_USER> /home/<MAIL_USER>/Maildir/
chmod -R 700 /home/<MAIL_USER>/Maildir/
```

---

## 6. Verification, Management, and Troubleshooting

### Service Management

Restart and enable Postfix and Dovecot services at boot.
```
systemctl restart postfix dovecot
systemctl enable postfix dovecot
systemctl status postfix dovecot
```

### Active Configuration Audit

Verify active parameters loaded into memory without syntax errors:
```bash
# Postfix active configuration audit
postconf -n

# Dovecot active configuration audit
doveconf -n
```

### TLS/SSL Connectivity Test

Verify certificate delivery over encrypted IMAPS (Port 993).
```bash
openssl s_client -connect <MAIL_FQDN>:993 -CAfile /etc/ssl/certs/ca-root.crt
```

### Queue Management and Diagnostics

Check current mail queue status:
```bash
postqueue -p
```

Force immediate queue delivery:
```bash
postqueue -f
```

Flush all messages from the mail queue:
```bash
postsuper -d ALL
```

Monitor live mail service transactions:
```bash
journalctl -u postfix -u dovecot -f
```

### Common Troubleshooting

- **SASL Authentication Failure (Error 535):** Verify that the unix socket `/var/spool/postfix/private/auth` exists with `0666` permissions and `postfix:postfix` ownership as configured in `10-master.conf`.

- **Relay Access Denied:** Ensure the client IP range is listed under `mynetworks` in `main.cf`, or verify that the mail client submits credentials via encrypted Port 587.


## 7. Advanced Features and Next Steps

To upgrade this mail infrastructure to an enterprise-grade production environment:

- **Email Authentication Standards:** Implement **SPF** (Sender Policy Framework), **DKIM** (DomainKeys Identified Mail), and **DMARC** records to prevent domain spoofing and spam classification.

- **Webmail Interface Integration:** Deploy **Roundcube** or **RainLoop** hosted on NGINX/Apache with PHP-FPM.

- **Virtual Users and Domains:** Migrate from system accounts (`/etc/passwd`) to virtual accounts backed by MariaDB/PostgreSQL or LDAP.

- **Antispam and Antivirus Filtering:** Integrate **Rspamd** or **Amavisd-new** with **ClamAV** and **SpamAssassin**.