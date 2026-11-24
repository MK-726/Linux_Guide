# Scheduling with cron, at, and batch

## Overview

Administrators schedule recurring jobs (backups, cleanups, reports) and one-time jobs. **cron** runs commands repeatedly on a schedule. **at** runs a command once at a given time. **batch** runs a command once when the system load is low.

## Key concepts

### The cron system

| Piece | Role |
|-------|------|
| `crond` (`cron` on Debian-based) | The daemon that wakes every minute and starts due jobs |
| **User crontabs** | One file per user, edited with `crontab -e`. Stored in `/var/spool/cron/` (RHEL-based: `/var/spool/cron/USER`, Debian-based: `/var/spool/cron/crontabs/USER`). **Do not edit those files directly** |
| `/etc/crontab` | System crontab. Has an **extra user field** |
| `/etc/cron.d/*` | System job files from packages and administrators. Same format as `/etc/crontab` (with user field) |
| `/etc/cron.hourly`, `cron.daily`, `cron.weekly`, `cron.monthly` | Directories with **executable scripts** run at those intervals |
| `anacron` | Runs missed daily/weekly/monthly jobs after downtime (for laptops and desktops). Configured in `/etc/anacrontab` |

### Crontab format

```
# ┌───────── minute (0-59)
# │ ┌─────── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌─── month (1-12 or jan-dec)
# │ │ │ │ ┌─ day of week (0-7, 0 and 7 = Sunday; or sun-sat)
# │ │ │ │ │
# * * * * *  command
```

| Symbol | Meaning | Example |
|--------|---------|---------|
| `*` | Every value | `* * * * *` every minute |
| `,` | List | `0 8,12,18 * * *` at 08:00, 12:00, 18:00 |
| `-` | Range | `0 9 * * 1-5` at 09:00 Monday to Friday |
| `/` | Step | `*/15 * * * *` every 15 minutes |
| `@reboot` | At startup | |
| `@hourly`, `@daily`, `@weekly`, `@monthly`, `@yearly` | Shortcuts | `@daily` = `0 0 * * *` |

### Examples

```
30 2 * * *       /usr/local/bin/backup.sh                  # every day at 02:30
0 */6 * * *      /usr/local/bin/sync.sh                    # every 6 hours
15 3 * * 0       /usr/bin/apt-get -qq update               # Sundays at 03:15
0 9 1 * *        /usr/local/bin/monthly-report.sh          # first day of the month at 09:00
*/5 * * * *      /usr/local/bin/check.sh >/dev/null 2>&1   # every 5 minutes, quiet
@reboot          /usr/local/bin/on-boot.sh
```

> [!NOTE]
> If you set **both** day-of-month and day-of-week (neither `*`), the job runs when **either** matches (logical OR), not both.

### The system crontab format (with user)

```
# /etc/cron.d/myapp
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=ops@example.com

# m h dom mon dow user    command
30 2 * * *  myapp  /opt/myapp/bin/cleanup.sh
```

## Commands

### Manage user crontabs

| Command | Purpose |
|---------|---------|
| `crontab -e` | Edit your crontab |
| `crontab -l` | List it |
| `crontab -r` | **Remove it without asking** |
| `crontab -u USER -e` | Edit another user's crontab (**root**) |
| `crontab file` | Install from a file |
| `crontab -l > cron.bak` | Back up |

> [!WARNING]
> `crontab -r` deletes your whole crontab immediately. The `r` key is next to `e`. Keep a backup (`crontab -l > ~/crontab.bak`) and consider using files in `/etc/cron.d/` managed by version control or automation.

### Environment of cron jobs

Cron runs jobs with a **minimal environment**, not your login shell:

| Variable | Typical value |
|----------|---------------|
| `PATH` | `/usr/bin:/bin` |
| `SHELL` | `/bin/sh` |
| `HOME` | The user's home |
| `LOGNAME`, `USER` | The user |

Set variables at the top of the crontab (`PATH=...`, `SHELL=/bin/bash`, `MAILTO=...`). Use **full paths** for commands and scripts.

### Output and mail

By default, output of a job is **mailed** to the owner (if a mail system exists). Control it:

```
MAILTO=ops@example.com                              # send output here ("" disables mail)
0 3 * * * /usr/local/bin/job.sh >> /var/log/job.log 2>&1   # log to a file
0 3 * * * /usr/local/bin/job.sh >/dev/null 2>&1            # discard (hides errors!)
```

Better: log to a file or the journal with `logger`: `... 2>&1 | logger -t myjob`.

### Percent signs

In a crontab, `%` means newline. Escape it:

```
0 1 * * * tar -czf /backup/data-$(date +\%F).tar.gz /data
```

### Special characters and multiple commands

Put complex logic in a **script**, not in the crontab line. Make it executable and test it by hand as the same user with the same minimal environment:

```console
$ env -i SHELL=/bin/sh PATH=/usr/bin:/bin /usr/local/bin/backup.sh
```

### Access control

| File | Effect |
|------|--------|
| `/etc/cron.allow` | If it exists, **only** listed users may use `crontab` |
| `/etc/cron.deny` | Listed users may **not** use `crontab` (if `cron.allow` does not exist) |

### Scripts in cron.daily and friends

Place an executable script (no `.sh` extension needed) in `/etc/cron.daily/`. On **Debian-based** systems `run-parts` runs only files whose names contain **letters, digits, underscores, and hyphens**. A name with a dot (`backup.sh`) can be skipped silently. Check with `run-parts --test /etc/cron.daily`.

The time these run is set in `/etc/crontab` (Debian-based) or by `anacron` / `cronie` configuration (`/etc/anacrontab`, RHEL-based).

### Logs

| System | Where to look |
|--------|---------------|
| Debian-based | `grep CRON /var/log/syslog` or `journalctl -u cron` |
| RHEL-based | `/var/log/cron` or `journalctl -u crond` |

```console
$ journalctl -u cron --since today | tail
Mar 03 02:30:01 server01 CRON[31502]: (myapp) CMD (/opt/myapp/bin/cleanup.sh)
```

The log shows **that** the job started, not whether it succeeded. Your job must log its own result.

### at: run once

Needs the `at` package and the `atd` service.

```console
$ at 14:30
at> /usr/local/bin/report.sh
at> <Ctrl+D>
job 7 at Mon Mar  3 14:30:00 2025

$ echo "/usr/local/bin/report.sh" | at now + 2 hours
$ at 02:00 tomorrow < /root/maintenance.txt
$ at 'midnight + 3 days'
```

| Command | Purpose |
|---------|---------|
| `at TIME` | Schedule a job (commands from stdin) |
| `atq` (or `at -l`) | List pending jobs |
| `at -c N` | Show the content of job N |
| `atrm N` (or `at -r N`) | Remove job N |

Time formats: `15:30`, `4pm`, `now + 30 minutes`, `noon tomorrow`, `10:00 AM Jul 31`.

`at` captures the current environment and working directory when you create the job. Control access with `/etc/at.allow` and `/etc/at.deny`.

### batch: run when the load is low

```console
$ echo "/usr/local/bin/reindex.sh" | batch
job 8 at Mon Mar  3 11:40:00 2025
```

`batch` waits until the system load average drops below a threshold (`atd -l`, about 0.8 by default on many systems), then runs the job. Useful for heavy tasks that should not compete with important work.

## Common pitfalls

- **"Works in my shell, fails in cron."** Cron has a minimal `PATH` and environment. Use full paths, set `PATH`, and test with `env -i`.
- **Unescaped `%`** cuts the command off at the first `%`.
- **`crontab -r` by accident.** Back up your crontabs.
- **No trailing newline** in a crontab file. The last line may be ignored. End every file with a newline.
- **Jobs overlapping.** A slow job started again by the next interval. Use `flock`: `*/5 * * * * flock -n /tmp/job.lock /usr/local/bin/job.sh`.
- **Silent failures.** `>/dev/null 2>&1` hides errors. Log and alert on failures.
- **Mail not configured.** Output is lost or fills `/var/mail`. Set `MAILTO` or redirect output.
- **Time zone and daylight saving.** Cron uses the system time zone. Jobs at 02:30 may skip or repeat when clocks change. Prefer UTC servers.
- **Editing `/var/spool/cron` directly.** Use `crontab -e`.
- **Files in `/etc/cron.d/` with wrong permissions or names.** They must be owned by root, not writable by others, and (Debian-based) use allowed characters in the name.
- **SELinux** may block jobs from doing something a normal shell can. Check `ausearch -m avc -ts recent`.
- **Cron in containers** needs the daemon running. Use the host's scheduler or a Kubernetes CronJob instead.

## Further reading

- [`crontab(5)`](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [`crontab(1)`](https://man7.org/linux/man-pages/man1/crontab.1.html)
- [`at(1)`](https://man7.org/linux/man-pages/man1/at.1.html)
- [`anacron(8)`](https://man7.org/linux/man-pages/man8/anacron.8.html)
- [crontab.guru](https://crontab.guru) (schedule expression checker)
