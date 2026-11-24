# Package Verification, Dependencies, and Update Management

## Overview

Installing is the easy part. Operating a system means checking that installed software is intact, handling dependency problems, and keeping systems patched in a planned way.

## Key concepts

### Verification levels

| Level | Question answered | Tool |
|-------|-------------------|------|
| **Signature** | Did this package come from a trusted source? | `rpm -K`, `gpgcheck`, APT signed repositories |
| **Checksum** | Was the file corrupted? | `sha256sum`, repository metadata |
| **Installed files** | Have files of an installed package been changed? | `rpm -V`, `debsums` |

## Commands

### Verify installed packages

**RHEL-based / SUSE-based**

```console
# rpm -V openssh-server           # no output = all files match
S.5....T.  c /etc/ssh/sshd_config
# rpm -Va | grep -v '^.......T.  c'   # whole system (slow); ignore only config-file changes
```

Reading the verify code:

| Letter | Meaning |
|--------|---------|
| `S` | Size differs |
| `5` | Checksum differs |
| `T` | Modification time differs |
| `M` | Mode (permissions) differs |
| `U` / `G` | Owner / group differs |
| `c` | The file is a configuration file |

Changes to configuration files (`c`) are normal. Changes to binaries (no `c`) need investigation.

**Debian-based**

```console
# apt install debsums
# debsums -s                      # show only changed or missing files
# debsums -c                      # list changed configuration files
# dpkg -V                         # built-in verify (newer dpkg)
```

### Check a package file before installing

```console
$ rpm -K package.rpm
package.rpm: digests signatures OK
$ dpkg-sig --verify package.deb           # only if the vendor signs .deb files
$ sha256sum -c SHA256SUMS                 # compare with the vendor's published list
```

### Dependency problems

| Problem | Action |
|---------|--------|
| Missing dependency | `apt install -f` (Debian), `dnf install` with the right repository enabled |
| Conflict between versions | `apt policy PKG`, `dnf repoquery`, check enabled repositories |
| Broken package state | `dpkg --configure -a`, `apt --fix-broken install`, `rpm --rebuilddb` (**root**) |
| What needs this package? | `apt-cache rdepends --installed PKG`, `dnf repoquery --installed --whatrequires PKG` |
| Why is this installed? | `apt-mark showmanual`, `aptitude why PKG`, `dnf history info`, `dnf repoquery --userinstalled` |

### Hold or lock versions

| Family | Command |
|--------|---------|
| Debian-based | `apt-mark hold PKG` / `unhold` |
| RHEL-based | `dnf versionlock add PKG` (plugin `python3-dnf-plugin-versionlock`), `dnf versionlock list` |
| SUSE-based | `zypper addlock PKG` |

Use holds sparingly and review them regularly, because held packages miss security fixes.

### Security updates only

```console
# apt-get -s upgrade | grep -i security           # simulate (Debian-based)
# unattended-upgrade --dry-run -d                 # Debian-based
# dnf upgrade --security                          # RHEL-based
# dnf updateinfo list security
# zypper patch --category security                # SUSE-based
```

### Do I need to reboot or restart services?

| Family | Command |
|--------|---------|
| Debian-based | `ls /var/run/reboot-required`, `needrestart` |
| RHEL-based | `dnf needs-restarting -r` (reboot needed?), `dnf needs-restarting -s` (services) |
| SUSE-based | `zypper ps -s`, `zypper needs-rebooting` |

Updating a library replaces the file on disk, but running programs keep the **old** version in memory until restarted. This matters for libraries such as OpenSSL and glibc.

### Automatic updates

| Family | Package / tool | Notes |
|--------|----------------|-------|
| Debian-based | `unattended-upgrades` | Configure `/etc/apt/apt.conf.d/50unattended-upgrades` and `20auto-upgrades`. Default is security updates only |
| RHEL-based | `dnf-automatic` | `/etc/dnf/automatic.conf`; enable `dnf-automatic.timer` |
| SUSE-based | `zypper` + systemd timer, or SUSE Manager | |

Example for RHEL-based systems:

```console
# dnf install dnf-automatic
# sed -i 's/^apply_updates.*/apply_updates = yes/' /etc/dnf/automatic.conf
# systemctl enable --now dnf-automatic.timer
```

### Rollback options

| Method | Where |
|--------|-------|
| `dnf history undo ID` | RHEL-based |
| Btrfs snapshots and `snapper rollback` | SUSE-based |
| LVM or VM snapshot before updating | Any |
| `apt install PKG=OLDVERSION` | Debian-based (if the old version is still in a repository or the cache) |
| Rebuild from automation (Ansible, images) | Any |

### Update process for servers (recommended outline)

1. Read the advisories and release notes.
2. Test the updates on a **staging** system that matches production.
3. Take a **snapshot or backup**.
4. Apply updates in a **maintenance window**, one node at a time for clusters.
5. Restart services or reboot where needed.
6. Verify the services (`systemctl --failed`, application checks, logs).
7. Document what changed.

## Common pitfalls

- **Auto-updating everything on production** without testing or rollback. Use automatic **security** updates only, and test the rest.
- **Forgetting to restart services** after a library update. The vulnerable code can still run.
- **Holding packages forever.** Track holds and review them.
- **Using `rpm --force` or `dpkg --force-all`.** It can damage the system. Fix the cause of the conflict instead.
- **Skipping the kernel reboot.** The old kernel keeps running. Use live-patch services (such as Canonical Livepatch, kpatch, or kGraft) when you cannot reboot, and still reboot during planned windows.
- **`rpm -Va` noise.** Expect config changes. Look for changed binaries and libraries.
- **Not checking disk space** before large upgrades (`/`, `/boot`, `/var`).

## Further reading

- [Debian: unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades)
- [DNF automatic](https://dnf.readthedocs.io/en/latest/automatic.html)
- [`rpm(8)`: verify](https://man7.org/linux/man-pages/man8/rpm.8.html)
- [Ubuntu Security Notices](https://ubuntu.com/security/notices)
- [Red Hat Security Advisories](https://access.redhat.com/security/security-updates/)
