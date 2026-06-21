# Homelab Journey

# 003 - Deploying AdGuard Home on Proxmox LXC

## Project Goal

The objective of this phase was to deploy a network-wide DNS filtering solution using AdGuard Home inside an LXC container.

The deployment was designed with the following principles:

* Lightweight virtualization using LXC.
* Network-wide ad and tracker blocking.
* Separation from other services.
* Easy recovery through Proxmox backups.
* Learning DNS and DHCP concepts through practical implementation.

---

# Starting Point

At the beginning of this phase the following infrastructure was already available:

## Existing Environment

### Host

* Proxmox VE 9.2
* GMKtec M5 Mini PC
* AMD Ryzen 7 7730U
* 32 GB RAM
* 512 GB NVMe SSD

### Existing Services

* Nextcloud running in CT100
* External 12 TB HDD
* DHCP reservation configured for Nextcloud
* Proxmox backup workflow already tested

---

# Learning Objectives

During this phase the objective was to understand:

* DNS
* DHCP
* DNS filtering
* AdGuard Home
* Upstream DNS servers
* DHCP reservations
* LXC containers
* Systemd services
* Service dependencies
* Network troubleshooting

---

# Why Deploy AdGuard Home?

A DNS server was selected as the next project because it introduces fundamental networking concepts.

Role:

```text
Client
   ↓
DNS Server
   ↓
Internet
```

Every device relies on DNS to translate domain names into IP addresses.

Example:

```text
google.com
   ↓
173.x.x.x
```

Without DNS most internet services become inaccessible.

---

# Downloading the Debian Template

Available templates were refreshed:

```bash
pveam update
```

Selected template:

```text
debian-13-standard_13.1-2_amd64.tar.zst
```

---

# LXC Container Creation

Container configuration:

```text
CT ID: 101
Hostname: adguard
Template: Debian 13
Unprivileged: Yes
CPU: 1 Core
RAM: 512 MB
Swap: 512 MB
Disk: 4 GB
```

Network:

```text
Bridge: vmbr0
IPv4: DHCP
IPv6: DHCP
Firewall: Enabled
```

---

# Why a Separate Container?

A decision was made to deploy AdGuard Home in its own container.

Advantages:

* Service isolation
* Easier troubleshooting
* Independent backups
* Easier future upgrades
* Reduced risk of affecting Nextcloud

Architecture:

```text
CT100 → Nextcloud
CT101 → AdGuard Home
```

---

# Installing AdGuard Home

The installation package was downloaded directly from AdGuard.

Installation process:

```bash
wget https://static.adguard.com/adguardhome/release/AdGuardHome_linux_amd64.tar.gz

tar xzf AdGuardHome_linux_amd64.tar.gz

cd AdGuardHome

./AdGuardHome -s install
```

---

# Understanding DNS

One important concept learned during deployment was the role of DNS.

Normal browsing:

```text
Browser
   ↓
google.com
   ↓
DNS Server
   ↓
Google IP Address
   ↓
Connection Established
```

Without DNS:

```text
google.com
   ↓
Unknown
```

The internet connection may still exist, but websites cannot be reached by name.

---

# Understanding DHCP

DHCP automatically provides:

* IP address
* Subnet mask
* Default gateway
* DNS servers

Example:

```text
IP Address:
192.168.0.134

Gateway:
192.168.0.1

DNS:
192.168.0.197
```

This allows network devices to function without manual configuration.

---

# Initial Configuration

The web setup wizard was accessed through:

```text
http://192.168.0.197:3000
```

Configuration:

```text
Web Interface:
Port 80

DNS Service:
Port 53

Listen Interface:
All Interfaces
```

After setup completion:

```text
http://192.168.0.197
```

became the permanent management interface.

---

# Understanding Ports

Several important ports were encountered:

```text
53    DNS
80    HTTP
3000  Initial AdGuard Setup
8006  Proxmox Web Interface
```

This phase reinforced the concept that multiple services can run on the same host by listening on different ports.

---

# DHCP Reservation

The container initially received:

```text
192.168.0.197
```

through DHCP.

A DHCP reservation was configured on the router.

Container MAC:

```text
BC:24:11:33:61:6C
```

Reserved IP:

```text
192.168.0.197
```

Result:

The container now receives a consistent address while still using DHCP.

---

# DNS Configuration

Router DHCP settings were updated.

Primary DNS:

```text
192.168.0.197
```

Secondary DNS:

```text
192.168.0.1
```

Purpose:

```text
AdGuard Running
     ↓
DNS Filtering Enabled

AdGuard Offline
     ↓
Router DNS Still Available
```

This prevents complete loss of internet access when the homelab server is powered off.

---

# Upstream DNS Servers

AdGuard itself does not know every internet address.

Instead it forwards requests to an upstream provider.

Selected upstream:

```text
Quad9 DNS-over-HTTPS
```

Architecture:

```text
Client
   ↓
AdGuard Home
   ↓
Quad9
   ↓
Internet
```

---

# Testing

DNS functionality was verified using:

```powershell
nslookup google.com 192.168.0.197
```

Successful resolution confirmed:

```text
Client
   ↓
AdGuard
   ↓
Internet
```

was functioning correctly.

Dashboard statistics also confirmed queries were being processed.

---

# Challenge #1 - Router Client Visibility

## Problem

The router detected:

```text
Nextcloud
```

but did not initially display:

```text
AdGuard
```

in the DHCP client list.

---

## Investigation

The service remained reachable:

```text
http://192.168.0.197
```

DNS queries were successfully processed.

This suggested a display issue rather than a network problem.

---

## Resolution

A manual DHCP reservation was created using the container MAC address.

Service functionality remained unaffected.

---

# Challenge #2 - DNS Dependency

## Problem

A DNS server becomes infrastructure.

If the server is offline:

```text
DNS Resolution
   ↓
Fails
```

which may appear as internet loss.

---

## Resolution

A secondary DNS server was configured on the router.

This allows internet access to continue even when the homelab server is turned off.

---

# Understanding DNS Filtering

AdGuard blocks requests at the DNS layer.

Example:

```text
Device
   ↓
Tracking Domain
   ↓
Blocked
```

This effectively blocks:

* Many advertisements
* Tracking domains
* Telemetry
* Malware domains

---

# Limitation - YouTube Ads

A significant discovery was that DNS filtering cannot reliably block advertisements inside the YouTube mobile application.

Reason:

```text
Video Content
Advertisements
```

are frequently served from the same domains.

Blocking those domains would also break video playback.

This is a limitation of DNS filtering rather than AdGuard itself.

---

# Backup

A Proxmox backup was created after successful deployment.

Generated file:

```text
vzdump-lxc-101-*.tar.zst
```

Purpose:

* Fast recovery
* Safe experimentation
* Configuration protection

---

# Service Startup

The container was configured to start automatically when the Proxmox host boots.

Purpose:

```text
Host Startup
    ↓
AdGuard Startup
    ↓
DNS Available
```

This minimizes service downtime after reboots.

---

# Current Architecture

```text
Internet
    |
 Quad9
    |
AdGuard Home
    |
Router
    |
Clients
```

Virtualization Layer:

```text
Proxmox Host
    |
CT100 - Nextcloud
CT101 - AdGuard Home
```

---

# Current Status

Completed:

* Debian 13 LXC deployment
* AdGuard Home installation
* DNS service configuration
* DHCP reservation
* Router DNS integration
* Upstream DNS configuration
* DNS testing
* Backup creation
* Automatic startup configuration

---

# Lessons Learned

Several key concepts became much clearer during this phase:

```text
DNS != DHCP
```

```text
Primary DNS != Upstream DNS
```

```text
Service Reachability != Router Visibility
```

```text
Internet Connection != DNS Resolution
```

```text
Backups > Reinstallation
```

The most important realization was that DNS is a critical infrastructure service.

Once a DNS server becomes part of the network, other services and devices begin to depend on it.

Design philosophy:

```text
Networking
↑
DNS
↑
Services
↑
Applications
```
