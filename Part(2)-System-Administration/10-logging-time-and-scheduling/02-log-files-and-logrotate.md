# Log Files, Locations, and Log Rotation with logrotate

## Overview

Log files grow forever unless something manages them. **logrotate** rotates, compresses, and deletes old logs on a schedule so disks do not fill up, and keeps a predictable history.

## Key concepts

### Where logs live: /var/log

| Path | Content |
|------|---------|
| `/var/log/syslog` (Debian-based) / `/var/log/messages` (RHEL-based, SUSE-based) | General system messages |
| `/var/log/auth.log` / `/var/log/secure` | Logins, `sudo`, SSH |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/dmesg` or `dmesg` command | Boot hardware messages |
| `/var/log/boot.log` | Boot messages |
| `/var/log/cron` (RHEL-based) | Cron activity |
| `/var/log/dpkg.log`, `/var/log/apt/` | Debian package activity |
| `/var/log/dnf.log`, `/var/log/yum.log` | RHEL package activity |
| `/var/log/audit/audit.log` | Auditing and SELinux (Chapter 16) |
| `/var/log/wtmp`, `btmp`, `lastlog` | Binary login records (`last`, `lastb`, `lastlog`) |
| `/var/log/journal/` | Persistent systemd journal |
| `/var/log/nginx/`, `/var/log/apache2/` or `httpd/`, `/var/log/mysql/` ... | Application logs |
| `/var/log/cloud-init.log` | Cloud instance initialization |

### How logrotate works

1. A scheduler runs `logrotate` once per day (a systemd timer `logrotate.timer` or `/etc/cron.daily/logrotate`).
2. `logrotate` reads its configuration and, for each log, decides if rotation is due.
3. It renames `app.log` to `app.log.1`, shifts older ones (`.1` → `.2`), compresses, and deletes the oldest.
4. It makes the application write to a **new** file.

```
app.log        (current)
app.log.1      (yesterday, not yet compressed with delaycompress)
app.log.2.gz   (older)
app.log.3.gz
```

## Configuration

| Path | Purpose |
|------|---------|
| `/etc/logrotate.conf` | Global defaults |
| `/etc/logrotate.d/*` | One file per application (installed by packages) |
| `/var/lib/logrotate/status` (Debian) or `/var/lib/logrotate/logrotate.status` (RHEL) | State: when each log was last rotated |

Example `/etc/logrotate.d/myapp`:

```
/var/log/myapp/*.log {
    daily
    rotate 14
    maxage 30
    missingok
    notifempty
    compress
    delaycompress
    dateext
    create 0640 myapp adm
    sharedscripts
    postrotate
        systemctl reload myapp.service > /dev/null 2>&1 || true
    endscript
}
```

### Common directives

| Directive | Meaning |
|-----------|---------|
| `daily`, `weekly`, `monthly` | Rotation interval |
| `size 100M` | Rotate when larger than this (instead of time) |
| `maxsize 100M` | Rotate when the time is due **or** the size is exceeded |
| `rotate N` | Keep N old logs |
| `maxage N` | Delete logs older than N days |
| `compress` / `nocompress` | Compress old logs (gzip by default; `compresscmd xz` also possible) |
| `delaycompress` | Compress one rotation later, so a program still writing to the old file is safe |
| `missingok` | No error if the log is missing |
| `notifempty` | Do not rotate empty logs |
| `create MODE USER GROUP` | Create the new log file with these permissions |
| `copytruncate` | Copy the log, then empty the original. Use when the program **cannot reopen** its log. Lines may be lost in between |
| `dateext` | Use dates (`app.log-20250303`) instead of numbers |
| `su USER GROUP` | Rotate as this user (needed when the directory is writable by non-root) |
| `sharedscripts` | Run `postrotate` once for all matching files, not for each |
| `prerotate` / `postrotate` ... `endscript` | Run commands before or after rotation |
| `olddir DIR` | Move old logs to another directory |

Which approach to use for the application to open the new file:

| Method | When |
|--------|------|
| `create` + signal or reload in `postrotate` | Best. The program reopens its log (Nginx: `nginx -s reopen` or `USR1`; rsyslog: `HUP`) |
| `copytruncate` | The program keeps the file open and cannot reopen |

## Commands

```console
# logrotate -d /etc/logrotate.d/myapp        # debug: dry run, show what would happen
# logrotate -v /etc/logrotate.conf           # verbose run
# logrotate -f /etc/logrotate.d/myapp        # force rotation now
$ cat /var/lib/logrotate/status | grep myapp
$ systemctl list-timers logrotate.timer
$ systemctl status logrotate.service
$ journalctl -u logrotate
```

```console
# logrotate -d /etc/logrotate.d/myapp
reading config file /etc/logrotate.d/myapp
considering log /var/log/myapp/app.log
  Now: 2025-03-03 06:25
  Last rotated at 2025-03-02 06:25
  log needs rotating
rotating log /var/log/myapp/app.log, log->rotateCount is 14
```

### Also manage logs by size in systemd

```console
# journalctl --vacuum-size=500M
# journalctl --vacuum-time=30d
```

## Common pitfalls

- **Program keeps writing to the old (renamed) file** after rotation. Use `postrotate` to reload, or `copytruncate`. Disk space is not freed until the program closes the old file (see [File Descriptors and Open Files](../../part-1-foundations/03-filesystem-and-file-management/07-file-descriptors-and-open-files.md)).
- **"error: skipping ... because parent directory has insecure permissions".** Add the `su user group` directive or fix the directory permissions.
- **Rotation never runs.** Check that `logrotate.timer` (or cron) is active and that the file is listed by `logrotate -d`.
- **SELinux denies `logrotate`** from writing to a non-standard log directory. Label the directory properly (`semanage fcontext`, `restorecon`).
- **Wildcards that match compressed files** (`/var/log/app/*`). Use `*.log`.
- **Daily rotation but too many files.** `rotate N` counts files, so `daily` with `rotate 7` keeps one week.
- **Rotating very large logs with compression** can use CPU and I/O. Use `nice`, or schedule off-peak.
- **Forgetting `create`** when the application cannot create its own log file.
- **Logs for compliance.** Check retention requirements (for example, 90 days to 1 year). Ship copies to central storage.

## Further reading

- [`logrotate(8)`](https://man7.org/linux/man-pages/man8/logrotate.8.html)
- [`logrotate.conf(5)`](https://man7.org/linux/man-pages/man5/logrotate.conf.5.html) (same page as above in some distributions)
- [Linux Filesystem Hierarchy: /var/log](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s10.html)
