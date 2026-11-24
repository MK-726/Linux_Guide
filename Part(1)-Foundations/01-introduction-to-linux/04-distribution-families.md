# Distribution Families

## Overview

Thousands of distributions exist, but most servers use a few **families**. Distributions in the same family share a package format, package manager, and many configuration habits. Learn the family and you can work on every distribution in it.

## Key concepts

### The three families in this guide

| Family | Package format | Package manager | Main distributions |
|--------|---------------|-----------------|--------------------|
| **Debian-based** | `.deb` | APT, dpkg | Debian, Ubuntu, Linux Mint |
| **RHEL-based** | `.rpm` | DNF, RPM | RHEL, Fedora, CentOS Stream, Rocky Linux, AlmaLinux |
| **SUSE-based** | `.rpm` | Zypper, RPM | SLES, openSUSE |

### Debian and Ubuntu

- **Debian** is a community project known for stability and a strong commitment to free software. It is the base for many other distributions.
- **Ubuntu** is built on Debian and developed by Canonical. It is very popular on cloud platforms and desktops. It offers **LTS** (Long Term Support) releases for production use.
- Package manager: `apt`. Firewall helper: `ufw`. Mandatory access control: AppArmor.

### RHEL, Fedora, and CentOS Stream

- **Red Hat Enterprise Linux (RHEL)** is a commercial distribution with a paid subscription, long support, and vendor certification. It is common in enterprises.
- **Fedora** is the community distribution sponsored by Red Hat. It moves fast and introduces new technology first.
- **CentOS Stream** is the community distribution that tracks just ahead of the next RHEL minor release. It is a development preview, not a stable rebuild of RHEL.
- Package manager: `dnf`. Firewall tool: `firewalld`. Mandatory access control: SELinux.

### Rocky Linux and AlmaLinux

After CentOS Linux (the old free rebuild of RHEL) ended, the community created **Rocky Linux** and **AlmaLinux** as free, RHEL-compatible distributions. They aim for binary compatibility with RHEL, so RHEL documentation and software usually apply directly.

### SUSE and openSUSE

- **SUSE Linux Enterprise Server (SLES)** is a commercial enterprise distribution, common in SAP and European environments.
- **openSUSE Leap** is a stable community release. **openSUSE Tumbleweed** is a rolling release.
- Package manager: `zypper`. Notable tools: **YaST** (configuration tool) and **Btrfs** with **snapper** snapshots by default.

### Other distributions you may meet

| Distribution | Notes |
|--------------|-------|
| Alpine Linux | Very small; common as a container base image |
| Arch Linux | Rolling release for experienced users |
| Amazon Linux | Optimized for AWS |
| Oracle Linux | RHEL-compatible, from Oracle |

## Commands

Detect the family from `/etc/os-release`. The `ID_LIKE` field names the parent family:

```console
$ grep -E '^(ID|ID_LIKE)=' /etc/os-release
ID=rocky
ID_LIKE="rhel centos fedora"
```

```console
$ grep -E '^(ID|ID_LIKE)=' /etc/os-release
ID=ubuntu
ID_LIKE=debian
```

Quick check of the package manager:

```console
$ command -v apt dnf zypper
/usr/bin/dnf
```

## Common pitfalls

- **Using the wrong package manager.** `apt` does not exist on RHEL-based systems, and `dnf` does not exist on Debian.
- **Assuming CentOS Stream is "CentOS Linux".** CentOS Stream is not a stable rebuild. Use Rocky Linux or AlmaLinux when you want a RHEL-compatible clone.
- **Following a tutorial for a different family.** Package names and file paths differ (for example, the Apache package is `apache2` on Debian-based and `httpd` on RHEL-based).

## Further reading

- [Debian](https://www.debian.org/doc/)
- [Ubuntu Server documentation](https://documentation.ubuntu.com/server/)
- [Red Hat documentation](https://docs.redhat.com)
- [Rocky Linux documentation](https://docs.rockylinux.org)
- [AlmaLinux wiki](https://wiki.almalinux.org)
- [SUSE documentation](https://documentation.suse.com)
- [openSUSE documentation](https://doc.opensuse.org)
