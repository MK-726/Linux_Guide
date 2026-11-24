# journalctl

## Overview

systemd collects logs from services, the kernel, and the boot process in the **journal**. `journalctl` reads it. Logs are structured (with metadata such as unit and PID) and indexed, so filtering is fast.

## Key concepts

- The journal is **binary**. You read it with `journalctl`, not `cat`.
- By default, many distributions store the journal in memory only (`/run/log/journal`, lost at reboot). Debian-based systems usually store it persistently. RHEL-based systems may not.
- Normal users see only their own messages unless they are in the `systemd-journal` (or `adm`, `wheel`) group.

## Commands

### Basic reading

| Command | Purpose |
|---------|---------|
| `journalctl` | Everything, oldest first |
| `journalctl -e` | Jump to the end |
| `journalctl -n 50` | Last 50 lines |
| `journalctl -f` | Follow new messages (like `tail -f`) |
| `journalctl -r` | Newest first |
| `journalctl --no-pager` | Plain output for pipes |
| `journalctl -o short-iso` | ISO timestamps |
| `journalctl -o json-pretty` | Full structured output |

### Filter

| Filter | Example |
|--------|---------|
| By unit | `journalctl -u nginx.service` |
| Several units | `journalctl -u nginx -u php-fpm` |
| By boot | `journalctl -b` (this boot), `journalctl -b -1` (previous boot) |
| By priority | `journalctl -p err` (errors and worse), `journalctl -p warning..err` |
| By time | `journalctl --since "2025-03-03 09:00" --until "2025-03-03 10:00"` |
| Relative time | `journalctl --since "1 hour ago"`, `--since yesterday` |
| Kernel messages | `journalctl -k` (like `dmesg`) |
| By PID | `journalctl _PID=812` |
| By user | `journalctl _UID=1000` |
| By program | `journalctl _COMM=sshd` |
| By message text | `journalctl -g "timeout"` (grep pattern) |
| By executable | `journalctl /usr/sbin/sshd` |

Priority levels: `0 emerg`, `1 alert`, `2 crit`, `3 err`, `4 warning`, `5 notice`, `6 info`, `7 debug`.

```console
$ journalctl -u nginx -p err --since today
Mar 03 10:12:01 server01 nginx[712]: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

$ journalctl -b -1 -p err           # errors in the previous boot (why did it crash?)
$ journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
 -1 5c1c...                          Sat 2025-03-01 08:00:11 UTC Mon 2025-03-03 08:55:02 UTC
  0 a77e...                          Mon 2025-03-03 09:00:05 UTC Mon 2025-03-03 11:30:00 UTC
```

Fields can be combined (AND):

```console
$ journalctl -u sshd -p warning --since "1 hour ago" --no-pager
```

### Disk usage and cleanup (root)

```console
# journalctl --disk-usage
Archived and active journals take up 312.0M in the file system.

# journalctl --vacuum-size=200M          # keep at most 200 MB
# journalctl --vacuum-time=14d           # delete older than 14 days
# journalctl --verify                    # check integrity
```

### Make the journal persistent

```console
# mkdir -p /var/log/journal
# systemd-tmpfiles --create --prefix /var/log/journal
# systemctl restart systemd-journald
```

Or set it in `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
SystemMaxUse=500M
MaxRetentionSec=1month
Compress=yes
```

```console
# systemctl restart systemd-journald
```

| `Storage=` value | Behavior |
|------------------|----------|
| `volatile` | Memory only (`/run`) |
| `persistent` | Disk (`/var/log/journal`) |
| `auto` | Persistent if `/var/log/journal` exists |
| `none` | Discard logs |

### Send your own message

```console
$ logger -t myscript "Backup finished"
$ journalctl -t myscript
$ systemd-cat -t myscript echo "hello"
```

For the larger logging picture (rsyslog, log files, logrotate) see Chapter 10.

## Common pitfalls

- **No logs from previous boots.** The journal is not persistent. Create `/var/log/journal` or set `Storage=persistent`.
- **"No journal files were found" / empty output for a normal user.** Add the user to `systemd-journal`, or use `sudo`.
- **Very large journal.** Set `SystemMaxUse=` and vacuum.
- **Output cut off at the screen width.** Use `--no-pager` or `-l`, or pipe to `less -S`.
- **Messages lost during very high log rates.** `RateLimitIntervalSec=` and `RateLimitBurst=` in `journald.conf` limit noisy services.
- **Time zone confusion.** Use `journalctl --utc` or check `timedatectl`.

## Further reading

- [`journalctl(1)`](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [`journald.conf(5)`](https://man7.org/linux/man-pages/man5/journald.conf.5.html)
- [`systemd.journal-fields(7)`](https://man7.org/linux/man-pages/man7/systemd.journal-fields.7.html)
