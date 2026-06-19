# Homelab Journey #002 - Deploying Nextcloud on Proxmox LXC

## Project Goal

The objective of this phase was to deploy a fully functional Nextcloud instance inside an LXC container while maintaining a clean separation between operating system files and user data.

The deployment was designed with the following principles:

* Lightweight virtualization using LXC.
* Data stored outside the container.
* Easy recovery through Proxmox backups.
* Learning Linux administration concepts through practical implementation.

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

### Storage

* External 12 TB HDD
* ext4 filesystem
* Mounted at:

```text
/mnt/storage
```

* Persistent mount configured through:

```text
/etc/fstab
```

---

# Learning Objectives

During this phase the objective was to understand:

* LXC containers
* Container networking
* Bind mounts
* Unprivileged containers
* Apache web server
* MariaDB
* Nextcloud architecture
* DHCP reservations
* Proxmox backups
* SMART monitoring

---

# Why LXC Instead of a Virtual Machine?

A decision was made to deploy Nextcloud inside an LXC container.

Advantages:

* Lower RAM usage
* Faster startup times
* Lower storage consumption
* Better resource efficiency

Tradeoffs:

* Shared kernel with the host
* Less isolation than a virtual machine

For a personal cloud deployment these tradeoffs were considered acceptable.

---

# Downloading the Debian Template

Available templates were refreshed:

```bash
pveam update
```

Available Debian templates:

```bash
pveam available | grep debian
```

Selected template:

```text
debian-13-standard_13.1-2_amd64.tar.zst
```

---

# LXC Container Creation

Container configuration:

```text
CT ID: 100
Hostname: nextcloud
Template: Debian 13
Unprivileged: Yes
Nesting: Enabled
CPU: 2 cores
RAM: 4096 MB
Swap: 1024 MB
Disk: 20 GB
```

Network:

```text
Bridge: vmbr0
IPv4: DHCP
IPv6: DHCP
Firewall: Enabled
```

---

# Understanding Host vs Container

One important concept learned during deployment was the distinction between:

```text
root@proxmox
```

and

```text
root@nextcloud
```

Commands executed inside the container cannot manage Proxmox resources.

For example:

```bash
pct set
```

only exists on the Proxmox host.

This distinction became important during storage configuration.

---

# Separating Application Data from User Data

A design decision was made to keep:

```text
Operating System
```

separate from:

```text
User Files
```

Container root filesystem:

```text
20 GB SSD
```

User data:

```text
12 TB HDD
```

This allows easier recovery and future migrations.

---

# Creating a Dedicated Data Directory

On the Proxmox host:

```bash
mkdir -p /mnt/storage/nextcloud-data
```

Purpose:

```text
/mnt/storage/nextcloud-data
```

would become the permanent storage location for all Nextcloud files.

---

# Bind Mount Configuration

A bind mount was added:

```bash
pct set 100 -mp0 /mnt/storage/nextcloud-data,mp=/data
```

This exposed the host directory:

```text
/mnt/storage/nextcloud-data
```

inside the container as:

```text
/data
```

Result:

```text
Host HDD
     ↓
Bind Mount
     ↓
Container
```

---

# Challenge #1 - Permission Denied

## Problem

Creating files inside:

```text
/data
```

produced:

```text
Permission denied
```

---

## Investigation

The container was configured as:

```text
Unprivileged LXC
```

which maps container root to UID:

```text
100000
```

on the host.

---

## Resolution

On Proxmox:

```bash
chown -R 100000:100000 /mnt/storage/nextcloud-data
```

After restarting the container:

```bash
touch /data/prueba.txt
```

succeeded.

---

# Installing Nextcloud Dependencies

Installed packages:

```bash
apt update && apt upgrade -y

apt install apache2 mariadb-server php php-gd php-mysql \
php-curl php-mbstring php-intl php-gmp php-bcmath \
php-xml php-zip php-imagick libapache2-mod-php \
unzip wget curl nano -y
```

---

# Apache

Apache was selected as the web server.

Role:

```text
Browser
   ↓
Apache
   ↓
Nextcloud
```

Apache serves the Nextcloud web application and processes incoming HTTP requests.

---

# MariaDB

MariaDB was installed as the database backend.

Purpose:

* User accounts
* Metadata
* File indexes
* Application configuration

---

# Challenge #2 - Database Credentials

## Problem

A placeholder password was accidentally used during user creation.

The installation wizard could not connect to the database.

---

## Investigation

Existing users were inspected:

```sql
SELECT User, Host FROM mysql.user;
```

Output confirmed:

```text
nextcloud@localhost
```

existed.

---

## Resolution

Password updated:

```sql
ALTER USER 'nextcloud'@'localhost'
IDENTIFIED BY 'new_password';

FLUSH PRIVILEGES;
```

Installation completed successfully.

---

# Nextcloud Data Directory

Instead of the default:

```text
/var/www/nextcloud/data
```

the following location was used:

```text
/data/nextcloud-data
```

which ultimately resides on:

```text
/mnt/storage/nextcloud-data
```

on the 12 TB HDD.

---

# Verifying Storage Location

Verification:

```bash
df -h /data
```

Output:

```text
Filesystem      Size
/dev/sda1        11T
```

This confirmed user files are stored on the HDD and not inside the container.

---

# DHCP Reservation

The container initially received:

```text
192.168.0.246
```

through DHCP.

A DHCP reservation was configured on the router.

Container MAC:

```text
BC:24:11:BF:9F:77
```

Reserved IP:

```text
192.168.0.246
```

Result:

The container now receives a consistent address while still using DHCP.

---

# Backups

A Proxmox backup was created.

Generated file:

```text
vzdump-lxc-100-*.tar.zst
```

Size:

```text
573 MB
```

Location:

```text
/var/lib/vz/dump
```

---

# Challenge #3 - Snapshot Limitation

## Problem

Snapshot creation was unavailable.

Error:

```text
The current guest configuration does not support taking new snapshots
```

---

## Root Cause

The container uses:

```text
mp0
```

bind-mounted storage.

This configuration prevents snapshot functionality.

---

## Resolution

Use standard Proxmox backups instead.

---

# System Health Checks

CPU:

```text
~52°C
```

NVMe SSD:

```text
44-50°C
```

12 TB HDD:

```text
49°C
```

SMART Status:

```text
PASSED
```

Verification commands:

```bash
sensors
smartctl -a /dev/sda
```

---

# Current Architecture

```text
Internet
    |
Router
    |
Proxmox Host
    |
LXC Debian 13
    |
Apache
    |
Nextcloud
    |
MariaDB
    |
Bind Mount
    |
12 TB HDD
```

---

# Current Status

Completed:

* Debian 13 LXC deployment
* Apache installation
* MariaDB installation
* Nextcloud installation
* Data separation
* Bind mount configuration
* DHCP reservation
* Backup creation
* SMART monitoring

---

# Lessons Learned

Several key concepts became much clearer during this phase:

```text
Host != Container
```

```text
Application Data != User Data
```

```text
Backups != Snapshots
```

```text
DHCP != DHCP Reservation
```

The most important realization was that the operating system and applications can always be rebuilt, while user data is the asset that must be protected first.

Design philosophy:

Data
↑
Backups
↑
Services
↑
Operating Systems
