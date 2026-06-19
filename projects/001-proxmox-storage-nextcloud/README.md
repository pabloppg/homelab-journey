# Homelab Journey #001 - Proxmox Storage Preparation for Nextcloud

## Project Goal

The objective of this project was to prepare a Proxmox VE host for a future Nextcloud deployment using LXC containers.

The long-term goal is to build a private cloud capable of:

* Automatically backing up photos from an iPhone.
* Storing data on a dedicated 12 TB hard drive.
* Learning Linux administration, storage management, networking, containers, and self-hosting concepts through hands-on practice.
* Understanding the reasoning behind every configuration step instead of simply following tutorials.

---

# Initial Environment

## Hardware

* GMKtec M5 Mini PC
* AMD Ryzen 7 7730U
* 32 GB RAM
* 512 GB NVMe SSD
* External 12 TB Western Digital HDD

## Software

* Proxmox VE 9.2
* Debian Trixie based host

---

# Learning Objectives

During this phase the objective was to understand:

* Linux storage concepts
* GPT partition tables
* Filesystems
* UUIDs
* Mount points
* Persistent mounts
* Linux repositories
* Proxmox repository management

---

# Repository Troubleshooting

## Problem

Running:

```bash
apt update
```

produced:

```text
401 Unauthorized
```

for:

```text
enterprise.proxmox.com
```

This happened because the default Proxmox installation was configured to use Enterprise repositories, which require a paid subscription.

---

## Investigation

APT repository files were inspected:

```bash
ls /etc/apt/sources.list.d
```

Output:

```text
ceph.sources
debian.sources
pve-enterprise.sources
```

Repository configuration was reviewed:

```bash
cat /etc/apt/sources.list.d/pve-enterprise.sources
```

Result:

```text
URIs: https://enterprise.proxmox.com/debian/pve
```

---

## Resolution

Enterprise repositories were disabled:

```bash
mv /etc/apt/sources.list.d/pve-enterprise.sources \
/etc/apt/sources.list.d/pve-enterprise.sources.disabled
```

Ceph Enterprise repository was also disabled:

```bash
mv /etc/apt/sources.list.d/ceph.sources \
/etc/apt/sources.list.d/ceph.sources.disabled
```

---

## Adding the Free Repository

A new repository definition was created:

```bash
nano /etc/apt/sources.list.d/pve-no-subscription.sources
```

Content:

```text
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

Verification:

```bash
apt update
```

Successful result:

```text
Proxmox VE updates
```

---

# Storage Preparation

## Existing Disk Layout

The external drive appeared as:

```bash
lsblk
```

Output:

```text
/dev/sda
└── /dev/sda1
```

The partition contained an exFAT filesystem.

---

# Concepts Learned

## Disk

Physical storage device:

```text
/dev/sda
```

---

## Partition

Logical subdivision of a disk:

```text
/dev/sda1
```

---

## Filesystem

Data organization structure:

Examples:

* FAT32
* exFAT
* NTFS
* ext4

---

## GPT

GUID Partition Table.

Stores information about:

* Partition boundaries
* Partition types
* Partition metadata

GPT does not store files.

It stores information about partitions.

---

# Rebuilding the Disk

## Inspecting Existing Layout

```bash
fdisk -l /dev/sda
```

Output:

```text
/dev/sda1
10.9T
Microsoft basic data
```

---

## Recreating the Partition

Entered fdisk:

```bash
fdisk /dev/sda
```

Operations performed:

```text
d
Delete existing partition

n
Create new partition

Accept default start sector

Accept default end sector

w
Write changes
```

Important learning:

Changes remain in memory until:

```text
w
```

is executed.

---

# Creating ext4

Before formatting:

```bash
blkid /dev/sda1
```

showed no filesystem information.

Filesystem creation:

```bash
mkfs.ext4 /dev/sda1
```

Result:

```text
Filesystem UUID:
36214931-a60a-480c-9723-ab72c7ab18c2
```

---

# Understanding UUIDs

A UUID uniquely identifies a filesystem.

Example:

```text
UUID=36214931-a60a-480c-9723-ab72c7ab18c2
```

This is preferred over:

```text
/dev/sda1
```

because device names may change between boots.

---

# Mounting the Disk

## Creating Mount Point

```bash
mkdir /mnt/storage
```

Important concept:

A mount point is simply a directory.

Before mounting:

```text
/mnt/storage
```

was an empty directory.

---

## Mounting

```bash
mount /dev/sda1 /mnt/storage
```

This connects:

```text
/dev/sda1
```

to:

```text
/mnt/storage
```

Analogy:

Windows:

```text
E:
```

Linux:

```text
/mnt/storage
```

---

## Verification

```bash
df -h /mnt/storage
```

Output:

```text
Filesystem      Size  Used Avail Use%
/dev/sda1        11T  2.1M   11T   1%
```

---

# Persistent Mounts

Problem:

Manual mounts disappear after reboot.

Solution:

Use:

```text
/etc/fstab
```

Filesystem Table.

Current entry added:

```text
UUID=36214931-a60a-480c-9723-ab72c7ab18c2 /mnt/storage ext4 defaults 0 2
```

Meaning:

```text
Filesystem:
UUID=36214931...

Mount Point:
/mnt/storage

Filesystem Type:
ext4

Options:
defaults
```

---

## Validation

```bash
mount -a
```

No errors indicated successful configuration.

Verification:

```bash
df -h /mnt/storage
```

Successful.

---

# Concepts Learned During This Project

* Linux directory hierarchy
* Difference between disks and partitions
* GPT partition tables
* Filesystems
* ext4
* UUIDs
* blkid
* fdisk
* mount
* fstab
* apt repositories
* Enterprise vs No-Subscription repositories
* Proxmox storage architecture

---

# Current Status

Completed:

* Proxmox installation
* Repository configuration
* HDD preparation
* ext4 filesystem creation
* Persistent storage configuration

Next Phase:

* Download Debian LXC template
* Create Debian container
* Connect container to storage
* Install Nextcloud
* Configure iPhone photo synchronization

---

# Lessons Learned

The most important realization was that Linux separates:

```text
Disk
↓
Partition
↓
Filesystem
↓
Mount Point
```

Understanding those four layers makes storage administration significantly easier and explains how Linux differs from the Windows drive-letter model.
