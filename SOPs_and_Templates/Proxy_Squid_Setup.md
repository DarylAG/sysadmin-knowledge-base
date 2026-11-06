# Standard Operating Procedure (SOP) - Squid Transparent Proxy with SSL Bumping

## Overview and Laboratory Topology

This Standard Operating Procedure (SOP) outlines the deployment of a **Squid Proxy server in Transparent Mode with SSL Bumping** within a simulated multi-node network infrastructure in GNS3.

In this environment, all servers and client hosts are interconnected through a central switch (`Switch1`) and routed to the external network/Internet via router `R1` and a `NAT1` cloud interface:

* **Topology Naming Scheme & Static IPs:**
  * `alpha-DDNS-1`: Dynamic DNS / DNS Server (`.1`)
  * `beta-CORREO-1`: Mail Server (`.2`)
  * `gamma-WEB&FTP-1`: Web & FTP Server (`.3`)
  * **`iota-PROXY-1`**: Squid Proxy Server (`.4`)
  * `delta-CA-1`: Internal Certificate Authority (CA)
  * `pcw01-1` / `pcl01-1`: Windows and Linux Client Workstations

<p align="center">
  <img src="../Assets/GNS3_LAB" alt="GNS3 Lab Topology" width="80%">
</p>

Since the DNS server is deployed directly into this pre-addressed topology, the initial system setup focuses on assigning its designated hostname (`alpha-DDNS-1`) and configuring its static IP parameters (`.1`) with an external DNS to download the necessary packages. **Once installed, the server is explicitly set to self-resolve DNS queries (using `127.0.0.1`) prior to configuring and integrating the core BIND9 and Kea services.**

---

## Conventions and Environment Variables

Replace the placeholder values in this document with your topology's specific settings:

* `<HOSTNAME>`: Server hostname (`iota-PROXY-1`).
* `<LOCAL_NETWORK>`: Subnet CIDR (e.g., `192.168.1.0/24` or matching your R1 LAN).
* `<PROXY_IP>`: Assigned static IP for the proxy server (`.4`).
* `<HTTP_PORT>`: Base listening port (`3128`).
* `<HTTP_INTERCEPT_PORT>`: HTTP transparent interception port (`3128`).
* `<HTTPS_PORT>`: HTTPS transparent interception port (`3129`).
* `<CA_NAME>`: Filename base of your CA certificate from `delta-CA-1` (e.g., `internal-ca`).

---

## Phase 1: System Preparation and Installation

### 1. Set Hostname
```bash
hostnamectl set-hostname <HOSTNAME>
```

### 2. Configure Static IP Address (via NetworkManager / nmcli)
```bash
# Example static IP configuration (adjust interface name and subnet to match your setup)
nmcli connection modify "eth0" ipv4.addresses 192.168.1.4/24 ipv4.gateway 192.168.1.254 ipv4.dns 192.168.1.1 ipv4.method manual
nmcli connection up "eth0"
```

### 3. Update Repositories and Install Packages
Once connectivity to the internet via R1 / NAT1 is confirmed
```bash
apt update && apt upgrade -y
apt install squid iptables squid-openssl -y
```

---

## Phase 2: SSL Certificate Management for SSL Bumping

To enable Squid to intercept encrypted HTTPS traffic, it must act as an internal Certificate Authority (CA) to generate certificates dynamically. It is mandatory to combine the public certificate and the unencrypted private key into a single  `.pem` file.

1. Create and navigate to the dedicated directory for storing Squid's SSL certificates
```bash
mkdir -p /etc/squid/certs
cd /etc/squid/certs/
```

2. Generate the unified `.pem` file by concatenating the public CA certificate (`.crt`) and the private key (`.key`):
```bash
cat <CA_NAME>.crt <CA_NAME>.key > <CA_NAME>.pem
```

3. Set strict security permissions on the certificate directory and `.pem` file to ensure only the proxy user has read access
```bash
chown -R proxy:proxy /etc/squid/certs
chmod 755 /etc/squid/certs
chmod 600 /etc/squid/certs/<CA_NAME>.pem
```

---

## Phase 3: SSL Database Initialization

Squid requires a dedicated directory to store dynamically generated certificates for inspected HTTPS websites.

1. Create the directory structure and assign ownership to the `proxy` user.
```bash
mkdir -p /var/lib/squid
chown -R proxy:proxy /var/lib/squid
```

2. Initialize the ssl_db database (allocated size: 4MB) and ensure the `proxy` user has full permissions over SSL storage:
```bash
/usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB
chown -R proxy:proxy /var/lib/squid/ssl_db
```


---
## Phase 4: Squid Configuration (`squid.conf`)

Create a safe backup of the default configuration file prior to making modifications:
```bash
mkdir -p /etc/squid/backup
cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
mv /etc/squid/*.bak /etc/squid/backup/
```

Open the clean `squid.conf` file:
```bash
nano /etc/squid/squid.conf
```

```conf
# ----------------------------------------------------------------------
# NETWORK DEFINITIONS AND SAFE PORTS
# ----------------------------------------------------------------------
acl localnet src <LOCAL_NETWORK>    # Defines local subnet range (e.g., 192.168.1.0/24)
acl Safe_ports port 80          # HTTP
acl Safe_ports port 443         # HTTPS
acl SSL_ports port 443          # HTTPS exclusive

# Deny requests to unsafe ports and non-SSL CONNECT attempts
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports

# ----------------------------------------------------------------------
# ACCESS CONTROL RULES (HTTP_ACCESS)
# ----------------------------------------------------------------------
http_access allow localhost
http_access allow localnet
http_access deny all

# ----------------------------------------------------------------------
# PORTS AND SSL BUMPING (TRANSPARENT PROXY)
# ----------------------------------------------------------------------
# Base proxy listening port (Squid 6+ requirement)
http_port <HTTP_PORT>

# Transparent HTTP interception port
http_port <HTTP_INTERCEPT_PORT> intercept

# Transparent HTTPS interception port with dynamic host certificate generation
https_port <HTTPS_PORT> intercept ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/certs/<CA_NAME>.pem

# SSL certificate database generator program
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 4MB

# SSL inspection steps (Peek and Bump)
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# ----------------------------------------------------------------------
# CACHE AND SYSTEM PARAMETERS
# ----------------------------------------------------------------------
maximum_object_size 4 MB
visible_hostname <HOSTNAME>
```
**Note:** *The `Safe_ports`directives, baseline network definitions, basic access controls, and maximum object size parameters are derived from the standard Squid baseline setup. The `intercept` and `ssl_bump` directives have been explicitly incorporated to satisfy the transparent proxy requirement for encrypted traffic inspection.*

---

## Phase 5: Firewall Rules (Iptables)

### Router Port Forwarding Configuration (R1)
> **Note:** For transparent interception to function across the network, the gateway router (`R1`) must forward incoming web traffic (TCP ports 80 and 443) targeting external networks to the proxy server.

To enable transparent proxying, redirect incoming client web traffic on standard HTTP and HTTPS ports to Squid's dedicated interception ports.
```bash
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port <HTTP_PORT>
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port <HTTPS_PORT>
```

## Phase 6: Service Management, Verification, and Troubleshooting

**1. Service Verification and Startup:** Validate the syntax of `/etc/squid/squid.conf` for fatal errors before starting or reloading the daemon
```bash
squid -k parse
```

Initialize the swap directory structure for local cache storage
```bash
squid -z
```

Restart the Squid daemon to apply changes and enable automatic startup on system boot
```bash
systemctl restart squid
systemctl enable squid
```

_Useful service management commands:_ `systemctl status squid`, `systemctl stop squid`.

**2. Log Monitoring and Code Interpretation:** Monitor real-time client requests, traffic flow, and operational events
```bash
tail -f /var/log/squid/access.log /var/log/squid/cache.log
```

- `TCP_MISS/200`: The requested resource was not found in the local cache. Squid fetched it successfully from the origin server and delivered it to the client.

- `TCP_DENIED/403`: The request was blocked by the proxy according to the defined  `http_access` or `acl` policies.


**3. SSL Database Repair (`ssl_db`):** If client machines experience unexpected SSL certificate warnings and log entries report `security_file_certgen` failures, reset and rebuild the dynamic SSL certificate database:
```bash
systemctl stop squid
rm -rf /var/lib/squid/ssl_db/*
/usr/lib/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB
chown -R proxy:proxy /var/lib/squid/ssl_db
systemctl start squid
```

---

## Next Steps and Advanced Configurations

Once the core transparent proxy setup is validated, consider expanding functionality with the following modules:

- **Deep Content Filtering:** Integrated _SquidGuard_ or _ufdbGuard_ to enforce domain blocking based on categories (e.g., social media, adult content, malware domains).

- **Granular Access Control Lists (ACLs):** Restrict access based on client MAC addresses or hardware signatures instead of IP subnets.

- **Bandwith Throttling (Delay Pools):** Define global or extension-specific download speed limits (e.g., throttling `.mp4` or `.iso` file transfers).

- **Time-Based Policies:** Scheduled network access rules to restrict specific domains or web services outside of business hours.