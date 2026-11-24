# Zypper and RPM (SUSE-based)

## Overview

On SUSE Linux Enterprise (SLES) and openSUSE, **zypper** is the high-level package manager and **rpm** is the low-level tool. Installing and removing needs **root**. Most `zypper` commands have short aliases.

## Commands

### Daily commands

| Command | Short | Purpose |
|---------|-------|---------|
| `zypper refresh` | `zypper ref` | Refresh repository metadata |
| `zypper install PKG` | `zypper in PKG` | Install |
| `zypper remove PKG` | `zypper rm PKG` | Remove |
| `zypper update` | `zypper up` | Update installed packages to newer versions |
| `zypper dist-upgrade` | `zypper dup` | Full distribution upgrade (**used for Tumbleweed** and release upgrades) |
| `zypper search WORD` | `zypper se WORD` | Search |
| `zypper info PKG` | `zypper if PKG` | Package details |
| `zypper search -i` | | Installed packages only |
| `zypper what-provides FILE` | `zypper wp FILE` | Which package provides a file or capability |
| `zypper clean` | `zypper cc` | Clear caches |

```console
# zypper refresh
# zypper install nginx
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 2 NEW packages are going to be installed:
  nginx nginx-module-xml
Overall download size: 612.4 KiB. Already cached: 0 B. After the operation, additional 1.5 MiB will be used.
Continue? [y/n/v/...? shows all options] (y):
```

### Patches (important on SUSE)

SUSE ships fixes as **patches**. Patches are grouped by category.

```console
# zypper list-patches                # all needed patches (zypper lp)
# zypper list-patches --category security
# zypper patch                       # install all needed patches
# zypper patch --category security   # only security patches
# zypper patch-check                 # exit code tells if patches are needed
```

### Repositories

| Command | Purpose |
|---------|---------|
| `zypper repos` (`lr`) | List repositories (`-u` shows URLs) |
| `zypper addrepo URL ALIAS` (`ar`) | Add a repository |
| `zypper removerepo ALIAS` (`rr`) | Remove a repository |
| `zypper modifyrepo --disable ALIAS` (`mr -d`) | Disable |
| `zypper modifyrepo --priority 90 ALIAS` | Change priority (lower number = higher priority) |
| `zypper --gpg-auto-import-keys refresh` | Accept new repository keys (use only for trusted repositories) |

On **SLES**, you register the system with `SUSEConnect` to get access to the official repositories (for example, `SUSEConnect -r REGISTRATION_CODE`).

### Locks and patterns

```console
# zypper addlock nginx              # prevent changes to a package (al)
# zypper locks                      # list locks (ll)
# zypper removelock nginx           # (rl)
# zypper search -t pattern          # patterns are groups of packages
# zypper install -t pattern devel_basis
```

### After updates: which services need a restart?

```console
# zypper ps -s
The following running processes use deleted files:

PID   | PPID | UID | User | Command | Service
------+------+-----+------+---------+--------
 812  | 1    | 0   | root | sshd    | sshd
```

`zypper ps -s` lists processes still using old libraries, so you know what to restart (or if a reboot is needed).

### rpm queries

The same `rpm` queries work as on RHEL-based systems: `rpm -qa`, `rpm -qi`, `rpm -ql`, `rpm -qf`, `rpm -V`. See [DNF and RPM](03-dnf-and-rpm.md).

### Transactional updates and snapshots

- openSUSE and SLES with **Btrfs** take **snapper** snapshots before and after `zypper` changes.
- `snapper list` shows snapshots. `snapper rollback` can revert the system.
- **SLE Micro / openSUSE MicroOS / Aeon / Kalpa** use **transactional-update**: updates are applied to a new snapshot, and take effect at the next reboot.

## Configuration files

| Path | Purpose |
|------|---------|
| `/etc/zypp/zypp.conf` | Main zypper/libzypp settings |
| `/etc/zypp/repos.d/*.repo` | Repository definitions |
| `/etc/zypp/locks` | Package locks |
| `/var/log/zypp/history` | Transaction log |
| `/var/cache/zypp/` | Cache |

## Common pitfalls

- **Using `zypper up` on Tumbleweed.** Rolling releases need `zypper dup`.
- **Mixing `zypper up` and `zypper patch`.** On SLES, `patch` is the normal way. `up` may install newer versions from third-party repositories.
- **Package vendor switches.** After adding Packman or OBS repositories, `zypper dup --allow-vendor-change` can switch packages to another vendor. Read the vendor-change list before confirming.
- **Not rebooting after kernel and core library updates.** Check `zypper ps -s` and `needs-restarting -r`.
- **Treating YaST as the only tool.** YaST is optional. Everything can be done with `zypper` and files.

## Further reading

- [openSUSE Zypper usage](https://en.opensuse.org/SDB:Zypper_usage)
- [SUSE documentation: managing software with zypper](https://documentation.suse.com)
- [`zypper(8)`](https://man.opensuse.org/zypper.8)
