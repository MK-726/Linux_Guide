# File Integrity Monitoring

## Overview

**File integrity monitoring (FIM)** detects unauthorized or unexpected changes to important files: system binaries, configuration, SSH keys, scheduled jobs, and boot files. Attackers modify files to hide, persist, or escalate. FIM compares the current state with a **trusted baseline** and reports differences. It answers *"what changed on this system, and was it supposed to?"*

## Key concepts

### How FIM works

1. **Baseline:** on a known-good system, compute a database of file attributes: permissions, owner, size, timestamps, and **cryptographic hashes** (SHA-256/512).
2. **Check:** periodically (or in real time) compare the files with the baseline.
3. **Report:** list added, removed, and changed files.
4. **Investigate and update:** decide whether changes are legitimate. If yes, update the baseline.

### Approaches

| Approach | Example tools | Notes |
|----------|---------------|-------|
| **Scheduled hash comparison** | **AIDE**, Tripwire, Samhain | Detects **what** changed. Simple and trusted |
| **Real-time monitoring** | `auditd` file watches, `inotify`-based tools, Wazuh/OSSEC syscheck | Detects **when, who, and how** (auditd gives the user and program) |
| **Package verification** | `rpm -Va`, `debsums`, `dpkg -V` | Compares files with the **package** database. Does not cover files outside packages |
| **Kernel integrity** | IMA/EVM | Measures and verifies files at run time (advanced) |
| **Immutable attributes** | `chattr +i` | Prevents changes to a few critical files |
| **Endpoint platforms** | Wazuh, Falco (runtime behavior), commercial EDR | Central management and alerts |

### What to monitor

| Monitor | Do not monitor (too noisy) |
|---------|---------------------------|
| `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`, `/lib*`, `/usr/lib*` | `/var/log`, `/var/cache`, `/var/tmp`, `/tmp` |
| `/etc` (especially `passwd`, `shadow`, `sudoers`, `ssh`, `pam.d`, `cron*`, `systemd`) | Databases and application data that change constantly |
| `/boot` (kernel, initramfs, GRUB) | `/proc`, `/sys`, `/dev`, `/run` |
| SSH keys: `~/.ssh/authorized_keys` | Home directory documents |
| Web server code (not uploads) | Mail spools |
| Kernel modules `/lib/modules` | |
| Systemd units, `ld.so.preload`, `/etc/profile.d` | |

## Commands

### AIDE (Advanced Intrusion Detection Environment)

**Install**

```console
# apt install aide aide-common         # Debian-based
# dnf install aide                     # RHEL-based
```

**Configuration:** `/etc/aide/aide.conf` and `/etc/aide/aide.conf.d/` (Debian-based), `/etc/aide.conf` (RHEL-based).

```
# Database locations
database_in=file:/var/lib/aide/aide.db.gz
database_out=file:/var/lib/aide/aide.db.new.gz
gzip_dbout=yes

# Define what attributes to check
MyRules = p+i+n+u+g+s+m+c+sha256+sha512+acl+xattrs+selinux
LogRules = p+u+g+n+S+acl+xattrs

# Paths
/boot         MyRules
/bin          MyRules
/sbin         MyRules
/usr          MyRules
/lib          MyRules
/etc          MyRules
/root/.ssh    MyRules
/var/log      LogRules

# Exclusions
!/etc/mtab
!/etc/resolv.conf
!/var/log/journal
!/usr/share/man
!/usr/src
```

| Attribute | Meaning |
|-----------|---------|
| `p` | Permissions |
| `i` | Inode |
| `n` | Number of links |
| `u`, `g` | Owner, group |
| `s` | Size |
| `S` | Size may only grow (logs) |
| `m`, `c`, `a` | mtime, ctime, atime (`a` is noisy) |
| `sha256`, `sha512` | Checksums |
| `acl`, `xattrs`, `selinux` | ACLs, extended attributes, SELinux label |

**Create the baseline** (on a clean, freshly patched system):

```console
# aide --init                                          # RHEL-based: creates /var/lib/aide/aide.db.new.gz
# mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
# aideinit                                             # Debian-based wrapper: creates the database in /var/lib/aide/
# cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db   # (Debian-based: file names without .gz)
```

**Check:**

```console
# aide --check                                         # RHEL-based; Debian-based: aide --check --config /etc/aide/aide.conf
AIDE found differences between database and filesystem!!

Summary:
  Total number of entries:      187432
  Added entries:                1
  Removed entries:              0
  Changed entries:              3

Added entries:
  /usr/local/bin/.hidden

Changed entries:
  /etc/passwd
  /usr/bin/ssh
  /etc/cron.d/backup
```

(Real reports also show a flag string for each path that tells you which attributes changed. The format varies by AIDE version.)

**Detailed view of one change:**

```console
# aide --check -V5 | less                              # verbose: shows which attributes changed (old and new values)
```

**Update the baseline** after you have **reviewed** that the changes are legitimate (for example, after patching):

```console
# aide --update                                        # writes aide.db.new(.gz); review again
# mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

**Schedule** (systemd timer or cron) and mail or ship the report:

```console
# cat /etc/cron.d/aide-check
30 4 * * * root /usr/sbin/aide --check 2>&1 | mail -s "AIDE report $(hostname)" security@example.com
```

Debian-based systems install a daily job (`/etc/cron.daily/aide`). RHEL-based systems do not, so create one.

### Protect the baseline

If an attacker can modify the AIDE database, the binary, or the configuration, they can hide changes.

- Store a **copy of the database off the host** or on **read-only media**, and compare checksums (`sha256sum /var/lib/aide/aide.db.gz`).
- Keep the `aide` binary and configuration checksums elsewhere.
- Run checks from a **trusted source** (for example a read-only mount or a remote orchestrator).
- Restrict permissions: database `600` owned by root.
- Update the baseline in a **controlled process**, ideally right after patching and deployments.

### Package verification as a quick FIM

```console
# rpm -Va 2>/dev/null | grep -v '^.......T.  c'      # RHEL-based / SUSE-based: show non-config changes
S.5....T.    /usr/bin/ssh                              # size, checksum, and mtime changed: suspicious unless updated by you
# debsums -s ; debsums -c                              # Debian-based
# dpkg -V                                              # Debian-based built-in verify
```

Only files that belong to packages are covered, and a compromised system can also tamper with the package database. Use it as a quick check.

### Real-time: who changed the file?

Combine FIM with **auditd** ([Linux Auditing and auditd](06-linux-auditing-and-auditd.md)):

```
-w /etc/ssh/sshd_config -p wa -k sshd
-w /usr/bin/ssh -p wa -k binaries
```

```console
# ausearch -k sshd -i
```

### Wazuh / OSSEC syscheck (agent-based)

Wazuh agents report changes in real time to a central manager and dashboards. Configuration in `ossec.conf`:

```xml
<syscheck>
  <frequency>43200</frequency>
  <directories check_all="yes" realtime="yes">/etc,/usr/bin,/usr/sbin</directories>
  <directories check_all="yes">/root/.ssh</directories>
  <ignore>/etc/mtab</ignore>
</syscheck>
```

### Tripwire and Samhain

**Tripwire** (open source and commercial) uses signed policy and database files. **Samhain** supports client-server operation and stealth mode. The principles are the same as AIDE.

### Handling the results

1. **Compare** the changes with your change records, deployments, and patching.
2. **Expected** changes: update the baseline.
3. **Unexpected** changes: treat as a **security incident**. Preserve evidence (copy the files, record hashes, keep logs), isolate the host if necessary, check for persistence (cron, systemd units, SSH keys, SUID files, new users), and rebuild from a known-good source when integrity cannot be trusted.

## Common pitfalls

- **Creating the baseline on an already compromised system.** Baseline after a clean install and patching.
- **Storing the database on the same writable disk** with no off-host copy.
- **Monitoring too much.** Noise leads to ignored alerts. Tune exclusions, and focus on critical paths.
- **Not updating the baseline after patching.** Every report becomes "changed" and people stop reading it. Automate: patch, review, `aide --update`.
- **Running checks as a heavy job at peak time.** Hashing the full system is I/O and CPU intensive. Schedule off-peak with `nice` and `ionice`.
- **No one reads the reports.** Send them to a monitored mailbox or SIEM, and alert on changes in critical files.
- **Relying on timestamps only.** Attackers can reset them. Use hashes.
- **Ignoring configuration drift** that is not an attack but an unauthorized change. FIM also supports change control.
- **Expecting FIM to prevent attacks.** It **detects**. Combine with MAC, least privilege, and patching.
- **Containers and immutable infrastructure.** Scan images and rebuild instead of monitoring files inside running containers.

## Further reading

- [AIDE manual](https://aide.github.io)
- [`aide.conf(5)`](https://man7.org/linux/man-pages/man5/aide.conf.5.html)
- [Wazuh: file integrity monitoring](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [Linux IMA/EVM](https://sourceforge.net/p/linux-ima/wiki/Home/)
- [Samhain](https://www.la-samhna.de/samhain/)
