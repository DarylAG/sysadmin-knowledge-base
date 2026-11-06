# Standard Operating Procedure (SOP) - Dynamic DNS (DDNS) with Kea DHCP and BIND9

## Overview and Laboratory Topology

This Standard Operating Procedure (SOP) outlines the deployment of an integrated **Dynamic DNS (DDNS) architecture using BIND9 and Kea DHCP** within a simulated multi-node network infrastructure in GNS3. 

In this environment, all servers and client hosts are interconnected through a central switch (`Switch1`) and routed to the external network/Internet via router `R1` and a `NAT1` cloud interface:

* **Topology Naming Scheme & Static IPs:**
  * **`alpha-DDNS-1`**: DNS & DHCP Server (`.1`) - *Primary target of this SOP*
  * `beta-CORREO-1`: Mail Server (`.2`)
  * `gamma-WEB&FTP-1`: Web & FTP Server (`.3`)
  * `iota-PROXY-1`: Squid Proxy Server (`.4`)
  * `delta-CA-1`: Internal Certificate Authority (CA)
  * `pcw01-1` / `pcl01-1`: Windows and Linux Client Workstations

<p align="center">
  <img src="../Assets/GNS3_LAB" alt="GNS3 Lab Topology" width="80%">
</p>

Since the DNS server is deployed directly into this pre-addressed topology, the initial system setup focuses on assigning its designated hostname (`alpha-DDNS-1`), configuring its static IP parameters (`.1`), and explicitly setting the server to self-resolve DNS queries (using `127.0.0.1`) prior to installing and integrating the BIND9 and Kea services.

---

## Conventions and Environment Variables

Replace the placeholder values in this document with your topology's specific settings:

* `<PRIMARY_HOSTNAME>`: Server hostname (e.g., `alpha-DDNS-1`).
* `<DOMAIN_NAME>`: Internal domain name (e.g., `xeon.cr`).
* `<NETWORK_INTERFACE>`: Primary network interface (e.g., `eth0` or `enp0s3`).
* `<SERVER_IP>`: Static IP address of the server (e.g., `192.168.1.1`).
* `<GATEWAY_IP>`: Gateway IP address (e.g., `192.168.1.254`).
* `<SUBNET_CIDR>`: Subnet in CIDR notation (e.g., `192.168.1.0/24`).
* `<REVERSE_ZONE_ARPA>`: Reversed network octets for the PTR zone (e.g., `1.168.192`).
* `<DHCP_RANGE>`: IP range for DHCP allocation (e.g., `192.168.1.100 - 192.168.1.199`).
* `<TSIG_KEY_NAME>`: Key name for DDNS updates (e.g., `dhcp1-alpha`).

---

## Phase 1: System Preparation and Network Configuration
Set the permanent hostname for the DNS/DHCP server node.
```bash
hostnamectl set-hostname <PRIMARY_HOSTNAME>
```

Configure the primary network interface with a static IP address and a reliable external DNS (`8.8.8.8`) to guarantee internet connectivity for the initial provisioning and package downloads
```bash
nmcli connection modify "<NETWORK_INTERFACE>" ipv4.addresses <SERVER_IP>/24 ipv4.gateway <GATEWAY_IP> ipv4.dns "8.8.8.8" ipv4.method manual
nmcli connection up "<NETWORK_INTERFACE>"
```

Update local repositories and install the BIND9 DNS.
```bash
apt update && apt upgrade -y
```

Install all core services (BIND9 for DNS and Kea for DHCP/DDNS integration) while the system still relies on its default external DNS resolution
```bash
# 1. Install BIND9 (DNS) packages
apt install -y bind9 bind9utils bind9-doc dnsutils

# 2. Install Kea dependencies and add the official ISC Kea repository
apt install curl apt-transport-https -y
curl -1sLf '[https://dl.cloudsmith.io/public/isc/kea-3-2/setup.deb.sh](https://dl.cloudsmith.io/public/isc/kea-3-2/setup.deb.sh)' | bash

# 3. Install Kea DHCPv4 and Kea DDNS servers
apt install isc-kea-dhcp4-server isc-kea-dhcp-ddns-server -y
```

Update the network configuration so the server strictly queries its own newly installed local BIND9 instance (`127.0.0.1`) for name resolution, relying on BIND9's internal forwarders for external internet queries.
```bash
nmcli connection modify "<NETWORK_INTERFACE>" ipv4.dns "127.0.0.1"
nmcli connection up "<NETWORK_INTERFACE>"
```

---

## Phase 2: BIND9 (DNS) Configuration

### 1. TSIG Key Generation

Generate a TSIG (Transaction Signature) cryptographic key to securely authenticate Dynamic DNS (DDNS) updates from the Kea DHCP server, and apply appropriate read permissions.
```bash
cd /etc/bind
tsig-keygen <TSIG_KEY_NAME> > <TSIG_KEY_NAME>.key
chmod 640 <TSIG_KEY_NAME>.key
```

### 2. Global Options Configuration

```bash
nano /etc/bind/named.conf.options
```

Configure BIND9 global settings by defining an ACL for trusted internal clients, enforcing recursion restrictions for security, defining listening ports, and establishing external forwarders for internet name resolution.
```conf
// Define trusted internal network scope
acl trusted {
        localhost;
        localnets;
        <SUBNET_CIDR>;
};

options {
        directory "/var/cache/bind";
        recursion yes;
        
        // Query and recursion permissions
        allow-query { trusted; };
        allow-query-cache { trusted; };
        allow-recursion { trusted; };

        // External DNS forwarders
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
        
        dnssec-validation auto;
        listen-on-v6 port 53 { ::1; };
        listen-on port 53 { 127.0.0.1; <SERVER_IP>; };
};
```

### 3. Local Zones and DDNS Policies Configuration

```bash
nano /etc/bind/named.conf.local
```

Define the authoritative forward and reverse zones for the local domain, and establish update policies (`update-policy`) that explicitly authorize the DHCP server to dynamically modify `A`, `PTR`, and `DHCID` records using the TSIG key.
```conf
// Include the generated TSIG key
include "/etc/bind/<TSIG_KEY_NAME>.key";

// Forward Zone
zone "<DOMAIN_NAME>" {
    type master;
    // Zones must reside in /var/lib/bind to grant the 'bind' user write permissions for DDNS
    file "/var/lib/bind/db.<DOMAIN_NAME>";
    update-policy {
        grant <TSIG_KEY_NAME> wildcard *.<DOMAIN_NAME> A DHCID;
    };
};

// Reverse Zone
zone "<REVERSE_ZONE_ARPA>.in-addr.arpa" {
    type master;
    file "/var/lib/bind/db.<REVERSE_ZONE_ARPA>";    
    update-policy {
        grant <TSIG_KEY_NAME> wildcard *.<REVERSE_ZONE_ARPA>.in-addr.arpa PTR DHCID;
   };
};
```

### 4. Zone Data Files Creation

Create the forward lookup zone file to map domain names to IP addresses, establishing the authoritative Start of Authority (SOA) and the primary Name Server (NS) records.
```bash
nano /var/lib/bind/db.<DOMAIN_NAME>
```

```conf
$TTL 604800	; 1 week
<DOMAIN_NAME>.		IN SOA	<PRIMARY_HOSTNAME>.<DOMAIN_NAME>. admin.<DOMAIN_NAME>. (
				2026110501 ; serial
				604800     ; refresh 
				86400      ; retry 
				2419200    ; expire 
				604800     ; minimum 
				)
			NS	<PRIMARY_HOSTNAME>.<DOMAIN_NAME>.
<PRIMARY_HOSTNAME>.<DOMAIN_NAME>.		A	<SERVER_IP>
```

Create the reverse lookup zone file to resolve IP addresses back to domain names (PTR records) for the defined internal subnet.
```bash
nano /var/lib/bind/db.<REVERSE_ZONE_ARPA>
```

```conf
$TTL 604800	; 1 week
<REVERSE_ZONE_ARPA>.in-addr.arpa. IN SOA <PRIMARY_HOSTNAME>.<DOMAIN_NAME>. admin.<DOMAIN_NAME>. (
				2026110501 ; serial
				604800     ; refresh 
				86400      ; retry 
				2419200    ; expire 
				604800     ; minimum 
				)
			NS	<PRIMARY_HOSTNAME>.<DOMAIN_NAME>.
1.<REVERSE_ZONE_ARPA>.in-addr.arpa. PTR	<PRIMARY_HOSTNAME>.<DOMAIN_NAME>.
```

### 5. Permissions and Service Restart

Assign proper ownership of the dynamic zone files to the `bind` user (allowing the service to write DDNS updates) and restart the BIND9 daemon to apply all configurations.
```bash
chown bind:bind /var/lib/bind/db.*
systemctl restart bind9
```

---

## Phase 3: Kea DHCP and DDNS Configuration

### 1. TSIG Key Extraction and Configuration

Extract the raw base64 secret string from the BIND9 TSIG key file and save it into a dedicated file for Kea.
```bash
cd /etc/kea
awk -F'"' '/secret/ {printf "%s", $2}' /etc/bind/<TSIG_KEY_NAME>.key > /etc/kea/<TSIG_KEY_NAME>.secret
```

Configure Kea to load the TSIG key definition, referencing the securely extracted secret file to authenticate DNS updates against BIND9.
```bash
nano /etc/kea/tsig-keys.json
```

```JSON
{
    "tsig-keys": [
        {
           "name": "<TSIG_KEY_NAME>",
           "algorithm": "hmac-sha256",
           "secret-file": "/etc/kea/<TSIG_KEY_NAME>.secret"
        }
    ]
}
```

Apply strict ownership and read permissions to the TSIG key configuration file, restricting access to the `_kea` service user.
```bash
chown _kea:_kea /etc/kea/tsig-keys.json
chmod 640 /etc/kea/tsig-keys.json
```

### 2. Service Configuration and Initialization

Configure the Kea DHCP-DDNS (D2) module settings, specifying how DHCP events trigger DNS update requests.
```bash
nano /etc/kea/kea-dhcp-ddns.conf
```

```JSON
{
  "DhcpDdns": {
    "ip-address": "127.0.0.1",
    "port": 53001,
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/var/run/kea/kea-ddns-ctrl-socket"
    },
    "forward-ddns": {
      "ddns-domains": [
        {
          "name": "<DOMAIN_NAME>.",
          "key-name": "<TSIG_KEY_NAME>",
          "dns-servers": [
            { "ip-address": "<SERVER_IP>" }
          ]
        }
      ]
    },
    "reverse-ddns": {
      "ddns-domains": [
        {
          "name": "<REVERSE_ZONE_ARPA>.in-addr.arpa.",
          "key-name": "<TSIG_KEY_NAME>",
          "dns-servers": [
            { "ip-address": "<SERVER_IP>" }
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

Configure the core Kea DHCPv4 server logic, including network subnets, IP allocation pools, and DDNS integration parameters.
```bash
nano /etc/kea/kea-dhcp4.conf
```

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": ["<NETWORK_INTERFACE>"]
    },
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv",
      "lfc-interval": 3600
    },
    "renew-timer": 15840,
    "rebind-timer": 27720,
    "valid-lifetime": 31680,
    "dhcp-ddns": {
      "enable-updates": true
    },
    "ddns-qualifying-suffix": "<DOMAIN_NAME>",
    "ddns-override-client-update": true,
    "subnet4": [
      {
        "id": 1,
        "subnet": "<SUBNET_CIDR>",
        "pools": [
          { "pool": "<DHCP_RANGE>" }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "<GATEWAY_IP>"
          },
          {
            "name": "domain-name-servers",
            "data": "<SERVER_IP>"
          },
          {
            "name": "domain-search",
            "data": "<DOMAIN_NAME>"
          }
        ]
        // Add reservations here
      }
      // Add subnets here
    ]
  }
}
```


Restart the Kea services to apply the new configurations and enable them to start automatically on system boot.
```bash
systemctl restart isc-kea-dhcp-ddns-server
systemctl restart isc-kea-dhcp4-server
systemctl enable isc-kea-dhcp-ddns-server isc-kea-dhcp4-server
```

---

## Phase 4: Management, Verification, and Troubleshooting

### Service Control Commands

Standard systemd commands to manage the lifecycle and verify the status of the DNS and DHCP services.
```bash
systemctl start/stop/restart/status bind9
systemctl start/stop/restart/status isc-kea-dhcp4-server
systemctl start/stop/restart/status isc-kea-dhcp-ddns-server
```

### Syntax Verification

Validate the syntax of the BIND9 configuration and zone files to detect errors prior to reloading the service.
```bash
named-checkconf
named-checkzone <DOMAIN_NAME> /var/lib/bind/db.<DOMAIN_NAME>
named-checkzone <REVERSE_ZONE_ARPA>.in-addr.arpa /var/lib/bind/db.<REVERSE_ZONE_ARPA>
```

Perform a dry-run syntax check (`-t`) on the Kea JSON configuration files to prevent service failures.
```bash
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
kea-dhcp-ddns -t /etc/kea/kea-dhcp-ddns.conf
```

### DDNS Zone Maintenance (BIND9)

Temporarily suspend dynamic updates (`freeze`) to safely modify a zone file manually, and subsequently resume normal DDNS operations (`thaw`).
```bash
rndc freeze <DOMAIN_NAME>
# Perform manual edits to the zone file...
rndc thaw <DOMAIN_NAME>
```

### Troubleshooting and Logs

Monitor real-time logs and operational events for the Kea DHCP service to diagnose allocation or DDNS issues.
```bash
journalctl -u isc-kea-dhcp4-server.service -e --no-pager
```

Inspect the local Kea lease database to verify active client IP assignments and lease expiration times.
```bash
more /var/lib/kea/kea-leases4.csv
```

## Phase 5: Advanced Features and Next Steps

- **DNSSEC Implementation:** Cryptographically sign the local zones to ensure the integrity, authenticity, and tamper-resistance of DNS responses.

- **High Availability (HA):** Configure a secondary (slave) BIND9 server utilizing zone transfers (`AXFR`), and deploy Kea DHCP's native High Availability hooks to establish active-passive or active-active redundancy.

- **Kea Database Backend:** Migrate the DHCP leases storage from an in-memory CSV file to a robust relational database engine (such as PostgreSQL or MySQL) for enhanced scalability in enterprise environments.