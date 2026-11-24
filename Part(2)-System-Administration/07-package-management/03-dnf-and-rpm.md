# DNF and RPM (RHEL-based)

## Overview

On RHEL, Fedora, CentOS Stream, Rocky Linux, and AlmaLinux, **DNF** is the high-level package manager (the command `yum` is a link to `dnf` on current releases). **RPM** is the low-level tool. Installing and removing needs **root**.

> [!NOTE]
> Fedora uses **DNF5** and RHEL 8/9 use **DNF 4**. Everyday commands are the same. Check the version with `dnf --version`.

## Commands

### Daily commands

```console
# dnf check-update                # list available updates (exit code 100 when updates exist)
# dnf upgrade                     # upgrade all packages
# dnf install nginx               # install
# dnf install nginx-1.24.0        # a specific version
# dnf remove nginx                # remove
# dnf autoremove                  # remove unneeded dependencies
# dnf reinstall nginx             # reinstall
# dnf downgrade nginx             # go back to the previous version
# dnf clean all                   # clear cached metadata
# dnf makecache                   # refresh metadata
```

DNF refreshes metadata automatically when it expires. There is no separate "update" step like `apt update`.

```console
# dnf install httpd
Dependencies resolved.
================================================================================
 Package         Arch     Version                 Repository          Size
================================================================================
Installing:
 httpd           x86_64   2.4.57-8.el9            appstream           47 k
Installing dependencies:
 apr             x86_64   1.7.0-12.el9            appstream          123 k
Transaction Summary
================================================================================
Install  9 Packages
Total download size: 2.1 M
Is this ok [y/N]:
```

`-y` answers yes automatically. Use it in scripts, not by habit.

### Search and information

| Command | Purpose |
|---------|---------|
| `dnf search WORD` | Search names and summaries |
| `dnf info PKG` | Details |
| `dnf list installed` | Installed packages |
| `dnf list available PKG` | Available versions |
| `dnf provides /usr/bin/dig` | Which package provides a file or command (also `dnf provides '*/dig'`) |
| `dnf repoquery --requires PKG` | Dependencies |
| `dnf repoquery --installed --whatrequires PKG` | What depends on it |
| `dnf repolist` | Enabled repositories (`--all` for all) |
| `dnf updateinfo list security` | Pending security errata |

### History and rollback

DNF records every transaction and can undo one.

```console
# dnf history
ID | Command line              | Date and time    | Action(s)      | Altered
 12 | install httpd            | 2025-03-03 10:12 | Install        |    9
 11 | upgrade                  | 2025-03-01 02:00 | I, U           |   34

# dnf history info 12
# dnf history undo 12          # undo transaction 12
```

### Groups and modules

```console
# dnf group list
# dnf group install "Development Tools"
# dnf module list nginx          # RHEL 8/9 application streams
# dnf module enable nginx:1.24
```

### Local packages

```console
# dnf install ./package.rpm       # resolves dependencies from repositories
```

### rpm: queries and low-level work

| Command | Purpose |
|---------|---------|
| `rpm -qa` | List all installed packages |
| `rpm -qa \| grep nginx` | Find installed packages |
| `rpm -qi PKG` | Package information |
| `rpm -ql PKG` | Files in the package |
| `rpm -qc PKG` | Configuration files of the package |
| `rpm -qf /usr/sbin/nginx` | Which package owns a file |
| `rpm -qR PKG` | What the package requires |
| `rpm -qp --list FILE.rpm` | Contents of a package file (not installed) |
| `rpm -V PKG` | Verify installed files against the database |
| `rpm -q --changelog PKG` | Changelog (check CVE fixes) |
| `rpm --import KEY` | Import a GPG key |
| `rpm -K FILE.rpm` | Check signature and digest |
| `rpm -Uvh FILE.rpm` | Upgrade or install a file (no repository dependency lookup) |
| `rpm -e PKG` | Erase |

## Configuration files

| Path | Purpose |
|------|---------|
| `/etc/dnf/dnf.conf` | Main DNF settings |
| `/etc/yum.repos.d/*.repo` | Repository definitions |
| `/etc/dnf/protected.d/` | Packages DNF will not remove |
| `/etc/pki/rpm-gpg/` | Repository GPG keys |
| `/var/log/dnf.log` (and `dnf.rpm.log`) | Logs |

Typical `dnf.conf` options:

```ini
[main]
gpgcheck=1
installonly_limit=3        # keep 3 kernels
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
fastestmirror=False
```

### Extra repositories on RHEL-based systems

| Repository | Purpose | Enable |
|------------|---------|--------|
| **AppStream, BaseOS** | Core content | Default |
| **CRB** (CodeReady Builder) / PowerTools (8) | Development libraries used by many packages | `dnf config-manager --set-enabled crb` |
| **EPEL** | Extra Packages for Enterprise Linux (community) | `dnf install epel-release` |

On **RHEL** itself, systems must be registered with `subscription-manager` to use Red Hat repositories.

## Common pitfalls

- **"Nothing provides ..." / dependency errors** after adding third-party repositories. Disable them (`--disablerepo=NAME`) and compare.
- **Looking for `apt update`.** DNF handles metadata refresh itself. Use `dnf makecache` to force it.
- **`rpm -ivh` on a local file** fails with missing dependencies. Use `dnf install ./file.rpm`.
- **Mixing EPEL with core packages** can cause upgrades to replace base packages. Use priorities or `excludepkgs=`.
- **Large transactions in the background.** `dnf-makecache.timer` or `dnf-automatic` may hold the lock. Wait and retry.
- **GPG check failures.** Import the correct key. Do not use `--nogpgcheck` as a habit.
- **Kernel updates need a reboot.** `dnf needs-restarting -r` (package `dnf-utils` or `yum-utils`) tells you if you must reboot.

## Further reading

- [DNF command reference](https://dnf.readthedocs.io/en/latest/command_ref.html)
- [`rpm(8)`](https://man7.org/linux/man-pages/man8/rpm.8.html)
- [Red Hat: managing software with DNF](https://docs.redhat.com)
- [EPEL](https://docs.fedoraproject.org/en-US/epel/)
