# IT Infrastructure & Systems Administration Documentation

Centralized technical repository containing standard operating procedures (SOPs), system hardening guides, and service configuration runbooks for enterprise-grade environments.

## Repository Architecture

* **`SOPs_and_Templates/`**: Production-ready configuration guides and step-by-step deployment procedures for core infrastructure services.
* **`Knowledge_Base/`**: Technical foundations, network architecture notes, and system administration references.

## Documented Services & Infrastructure

* **Directory Services & Authentication:** OpenLDAP (`slapd`) directory structure configuration.
* **Network Infrastructure:** BIND9 DNS master/slave synchronization, Kea DHCP integration, and IP address management.
* **Perimeter & Security:** Squid Proxy with SSL Bumping (Man-in-the-Middle traffic inspection) and local OpenSSL Public Key Infrastructure (PKI).
* **Storage Protocols:** Network File System (NFS) and Samba (SMB) centralized file shares.
* **Mail & Web Services:** Postfix and Dovecot mail servers, Apache2 Virtual Hosts, Nginx web servers, and FTP (`vsftpd` / `ProFTPD`) services.
* **Backup & Recovery:** Bacula enterprise backup server integration.

## Documentation Standards

All manuals follow a **"Docs as Code"** methodology, integrating complete configuration templates, syntax-highlighted code blocks, and deployment steps directly within markdown files to ensure clarity and rapid execution.

## Purpose

Maintained as a professional proof-of-work portfolio demonstrating hands-on competencies in Linux server administration, network routing, and defensive cybersecurity operations.