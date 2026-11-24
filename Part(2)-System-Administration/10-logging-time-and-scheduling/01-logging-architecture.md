# Linux Logging Architecture: systemd-journald, rsyslog, and syslog

## Overview

Logs are the first place to look when something goes wrong, and the main evidence for security and audits. Linux has two logging systems that usually work **together**:

- **systemd-journald** collects logs from the kernel, services, and boot into a structured **binary journal**.
- **rsyslog** (the classic **syslog** daemon) writes **text files** in `/var/log` and can forward logs to other servers.

## Key concepts

### How the pieces fit

```
Kernel messages ──┐
systemd services ─┤ stdout/stderr
Applications ─────┤ syslog() calls / /dev/log
                  ▼
          systemd-journald  ─────►  journal (/run/log/journal or /var/log/journal)
                  │ (forward)             read with journalctl
                  ▼
              rsyslog  ───────────►  /var/log/*.log  (text files)
                  └───────────────►  remote log server (TCP/UDP/TLS)
```

| Component | Role |
|-----------|------|
| **journald** | Receives all logs, adds metadata (unit, PID, UID, time), indexes them |
| **rsyslog** | Filters by facility and severity, writes text files, forwards over the network. Reads from the journal (`imjournal`) or from the socket |
| **`/dev/log`** | The local syslog socket applications use |
| **Application log files** | Programs like Nginx and PostgreSQL may write their own files, independent of syslog |

> [!NOTE]
> Minimal installs may have **only the journal**. For example, recent Debian does not install rsyslog by default, so `/var/log/syslog` and `/var/log/auth.log` do not exist until you install it. Use `journalctl` instead.

### Syslog facilities

A **facility** says what kind of program sent the message.

| Facility | Used by |
|----------|---------|
| `kern` | Kernel |
| `user` | User-level programs |
| `mail` | Mail system |
| `daemon` | System daemons |
| `auth`, `authpriv` | Authentication and security (`sshd`, `sudo`, `login`). `authpriv` is private |
| `syslog` | The syslog daemon itself |
| `cron` | Scheduler |
| `lpr`, `news`, `uucp`, `ftp` | Legacy services |
| `local0` – `local7` | Free for your own applications |

### Syslog severities

| Level | Name | Meaning |
|-------|------|---------|
| 0 | `emerg` | System is unusable |
| 1 | `alert` | Act immediately |
| 2 | `crit` | Critical |
| 3 | `err` | Error |
| 4 | `warning` | Warning |
| 5 | `notice` | Normal but significant |
| 6 | `info` | Informational |
| 7 | `debug` | Debug |

A rule for severity `warning` includes `warning` and **more severe** levels (0–4).

### Anatomy of a log line

```
Mar  3 10:15:22 server01 sshd[2100]: Failed password for invalid user admin from 203.0.113.9 port 51234 ssh2
└─ time ──────┘ └ host ┘ └ program[PID] ┘└────────────── message ─────────────────────────────────────┘
```

## Commands and configuration

### journald

Configuration: `/etc/systemd/journald.conf` and `/etc/systemd/journald.conf.d/*.conf`. See [journalctl](../06-boot-grub-systemd/06-journalctl.md) for reading and sizing the journal.

```ini
[Journal]
Storage=persistent
SystemMaxUse=1G
ForwardToSyslog=yes
```

### rsyslog

| File | Purpose |
|------|---------|
| `/etc/rsyslog.conf` | Main configuration |
| `/etc/rsyslog.d/*.conf` | Your additions (read in alphabetical order) |

Rule format: `selector  action`, with `facility.severity` as the selector.

```
# /etc/rsyslog.d/10-custom.conf
auth,authpriv.*              /var/log/auth.log
*.*;auth,authpriv.none       /var/log/syslog
kern.*                       /var/log/kern.log
mail.err                     /var/log/mail.err
local3.*                     /var/log/myapp.log
*.emerg                      :omusrmsg:*
```

| Part | Meaning |
|------|---------|
| `auth,authpriv.*` | All severities of two facilities |
| `*.*;auth,authpriv.none` | Everything except those facilities |
| `mail.err` | Severity `err` and higher |
| `-/var/log/file` | A leading `-` (old syntax) meant "do not sync after each write" |
| `:omusrmsg:*` | Send to all logged-in users |

Apply and test:

```console
# rsyslogd -N1                   # check syntax
# systemctl restart rsyslog
$ logger -p local3.info "test message"
$ tail -1 /var/log/myapp.log
```

### Remote logging (centralized logging)

Send all logs to a central server so an attacker on one host cannot erase their traces, and so you can search in one place.

**Client** (`/etc/rsyslog.d/90-forward.conf`):

```
*.*  action(type="omfwd" target="loghost.example.com" port="514" protocol="tcp"
            action.resumeRetryCount="100" queue.type="LinkedList" queue.size="10000")
```

The short form `*.* @@loghost.example.com:514` uses TCP (`@@`). A single `@` uses UDP, which can lose messages.

**Server** (`/etc/rsyslog.d/10-server.conf`):

```
module(load="imtcp")
input(type="imtcp" port="514")
```

Open TCP port 514 in the firewall, restrict it to trusted clients, and use **TLS** (`gtls` / `imtcp` with certificates) over untrusted networks. For larger environments, centralized stacks (ELK/OpenSearch, Loki, Graylog) are used (Chapter 25).

### Writing your own log messages

```console
$ logger "Backup started"
$ logger -p local3.warning -t mybackup "Disk almost full"
$ logger -t myscript -f /tmp/output.txt          # log a file's content
```

In scripts, log with `logger` so messages reach both the journal and `/var/log`.

### Distribution differences

| | Debian-based | RHEL-based | SUSE-based |
|---|-------------|------------|------------|
| General log | `/var/log/syslog` | `/var/log/messages` | `/var/log/messages` |
| Authentication log | `/var/log/auth.log` | `/var/log/secure` | `/var/log/messages` (also journal) |
| Default journal storage | Persistent (Debian, Ubuntu) | Often volatile; check | Persistent |
| rsyslog installed by default | Ubuntu: yes. Debian: no (recent) | Yes | Not always. Journal is primary |

## Common pitfalls

- **Looking for `/var/log/messages` on Debian-based systems** (it is `/var/log/syslog`).
- **Rsyslog not installed**, so text log files are missing. Install `rsyslog` or use `journalctl`.
- **Logs only in memory.** After reboot the journal is gone when `Storage=volatile`. Make it persistent.
- **UDP log forwarding** loses messages. Use TCP (or RELP) with queues.
- **Duplicate messages** when both `ForwardToSyslog` and `imjournal` are active. Use one path.
- **Clocks out of sync** make logs from several hosts impossible to correlate. See [NTP and chrony](04-time-zones-ntp-and-chrony.md).
- **Logging sensitive data** (passwords, tokens) into logs. Review application log settings.
- **Not protecting logs.** Restrict permissions and ship them off the host.

## Further reading

- [`rsyslog.conf(5)`](https://man7.org/linux/man-pages/man5/rsyslog.conf.5.html)
- [rsyslog documentation](https://www.rsyslog.com/doc/)
- [`journald.conf(5)`](https://man7.org/linux/man-pages/man5/journald.conf.5.html)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
