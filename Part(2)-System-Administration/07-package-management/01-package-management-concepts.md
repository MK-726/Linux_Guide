# Package Management Concepts

## Overview

On Linux you rarely download an installer from a website. You use a **package manager**, which installs software from trusted **repositories**, handles **dependencies**, and tracks every file it installs. This makes installation, updates, and removal consistent and safe.

## Key concepts

### What is a package?

A **package** is an archive that contains:

- The program files and their target paths
- **Metadata**: name, version, description, dependencies
- Scripts that run before or after install or removal
- Checksums and a signature

| Family | Package file | Low-level tool | High-level tool |
|--------|-------------|----------------|-----------------|
| Debian-based | `.deb` | `dpkg` | `apt`, `apt-get` |
| RHEL-based | `.rpm` | `rpm` | `dnf` (formerly `yum`) |
| SUSE-based | `.rpm` | `rpm` | `zypper` |

### Two layers

| Layer | Job | Resolves dependencies? |
|-------|-----|------------------------|
| **Low-level** (`dpkg`, `rpm`) | Install, remove, and query **one package file** | **No** |
| **High-level** (`apt`, `dnf`, `zypper`) | Download from repositories, resolve dependencies, install everything needed | **Yes** |

Use the high-level tool in daily work. Use the low-level tool for queries (what files, which package owns this file) and rare repairs.

### Repositories

A **repository** (repo) is a server that holds packages and an **index** (list of packages and versions). The package manager downloads the index, finds the newest version, and downloads what you ask for. Repositories are **signed** with GPG keys so the system can check authenticity.

### Dependencies

Most programs need libraries or other programs. A package lists them as dependencies. The package manager installs them automatically. If a dependency cannot be met, the install fails with a clear error. This is better than "DLL hell", but conflicts between repositories can still happen.

### Where data is stored

| | Debian-based | RHEL-based / SUSE-based |
|---|-------------|--------------------------|
| Installed package database | `/var/lib/dpkg/` | `/var/lib/rpm/` (or `/usr/lib/sysimage/rpm/` on newer systems) |
| Downloaded packages (cache) | `/var/cache/apt/archives/` | `/var/cache/dnf/` or `/var/cache/zypp/` |
| Repository configuration | `/etc/apt/` | `/etc/yum.repos.d/`, `/etc/zypp/repos.d/` |
| Package manager logs | `/var/log/apt/`, `/var/log/dpkg.log` | `/var/log/dnf.log`, `/var/log/zypp/history` |

### Package states

- **Installed:** files on disk and recorded in the database.
- **Removed:** program files deleted. Configuration may remain (Debian `remove`) or be deleted (Debian `purge`).
- **Held / locked:** pinned so it will not upgrade.

### Package naming

```
nginx-1.24.0-2.el9.x86_64.rpm
 │     │      │   │    └── architecture
 │     │      │   └─────── distribution tag
 │     │      └─────────── release (packaging revision)
 │     └────────────────── upstream version
 └──────────────────────── name

openssh-server_1:9.6p1-3ubuntu13_amd64.deb
```

### Equivalent commands

| Task | APT | DNF | Zypper |
|------|-----|-----|--------|
| Refresh repository data | `apt update` | `dnf makecache` (done automatically) | `zypper refresh` |
| Install | `apt install PKG` | `dnf install PKG` | `zypper install PKG` |
| Remove | `apt remove PKG` | `dnf remove PKG` | `zypper remove PKG` |
| Update all | `apt upgrade` | `dnf upgrade` | `zypper update` |
| Search | `apt search WORD` | `dnf search WORD` | `zypper search WORD` |
| Package info | `apt show PKG` | `dnf info PKG` | `zypper info PKG` |
| Which package owns a file | `dpkg -S FILE` | `rpm -qf FILE` | `rpm -qf FILE` |
| Files of a package | `dpkg -L PKG` | `rpm -ql PKG` | `rpm -ql PKG` |
| List installed | `apt list --installed` | `dnf list installed` | `zypper search -i` |
| Clean cache | `apt clean` | `dnf clean all` | `zypper clean` |

## Common pitfalls

- **Using `rpm -i` or `dpkg -i` and getting dependency errors.** Use `dnf install ./file.rpm` or `apt install ./file.deb` so dependencies are resolved.
- **Mixing repositories from different releases.** Adding a repo built for another distribution version can break the system.
- **Two package managers on one system** (for example, installing the same software through the package manager and from a tarball). Keep one source per application.
- **Running two package commands at the same time.** The database is locked. Wait for the other process.
- **Not reading what a command will do.** Read the transaction summary (packages to be removed!) before you confirm.

## Further reading

- [Debian Reference: package management](https://www.debian.org/doc/manuals/debian-reference/ch02.en.html)
- [DNF documentation](https://dnf.readthedocs.io)
- [openSUSE Zypper cheat sheet](https://en.opensuse.org/SDB:Zypper_usage)
