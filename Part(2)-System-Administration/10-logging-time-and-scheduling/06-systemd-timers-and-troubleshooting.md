# systemd Timers and Troubleshooting Scheduled Jobs

## Overview

A **systemd timer** is a unit that starts another unit (usually a service) on a schedule. Timers are the modern alternative to cron. They give you logging in the journal, dependencies, resource limits, catch-up of missed runs, and clear status. This page also covers how to troubleshoot jobs that do not run.

## Key concepts

### A timer needs two units

| Unit | Role |
|------|------|
| `NAME.service` | **What** to run (the job) |
| `NAME.timer` | **When** to run it |

By default a timer starts the service with the same name.

### Timer types

| Type | Directive | Meaning |
|------|-----------|---------|
| **Realtime (calendar)** | `OnCalendar=` | At a wall-clock time, like cron |
| **Monotonic** | `OnBootSec=` | A time after boot |
| | `OnStartupSec=` | After systemd started |
| | `OnUnitActiveSec=` | After the service last became active (interval) |
| | `OnUnitInactiveSec=` | After the service last finished |

### Important options

| Option | Meaning |
|--------|---------|
| `Persistent=true` | If the system was off at the scheduled time, **run the missed job at the next start** (like anacron). Only for `OnCalendar=` |
| `RandomizedDelaySec=15min` | Spread start times randomly, to avoid load spikes on many hosts |
| `AccuracySec=1min` | Allowed delay for power efficiency (default 1 minute) |
| `Unit=other.service` | Start a different service |
| `WakeSystem=true` | Wake the machine from suspend |

### OnCalendar syntax

Format: `DayOfWeek Year-Month-Day Hour:Minute:Second`

| Expression | Meaning |
|------------|---------|
| `hourly`, `daily`, `weekly`, `monthly` | Shortcuts (daily = `*-*-* 00:00:00`) |
| `*-*-* 02:30:00` | Every day at 02:30 |
| `Mon..Fri *-*-* 09:00:00` | Weekdays at 09:00 |
| `*-*-01 04:00:00` | First of every month at 04:00 |
| `*:0/15` | Every 15 minutes |
| `Sun *-*-* 03:00:00` | Every Sunday at 03:00 |
| `*-*-* 02:30:00 UTC` | At 02:30 UTC |

Test any expression:

```console
$ systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
  Original form: Mon..Fri *-*-* 09:00:00
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Tue 2025-03-04 09:00:00 UTC
       (in UTC): Tue 2025-03-04 09:00:00 UTC
       From now: 21h left
$ systemd-analyze calendar --iterations=5 "*:0/15"
```

## Commands

### Example: nightly backup timer

`/etc/systemd/system/backup.service`:

```ini
[Unit]
Description=Nightly backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup.sh
Nice=10
IOSchedulingClass=idle
```

`/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Run backup every night

[Timer]
OnCalendar=*-*-* 02:30:00
RandomizedDelaySec=10min
Persistent=true

[Install]
WantedBy=timers.target
```

Activate (enable the **timer**, not the service):

```console
# systemctl daemon-reload
# systemctl enable --now backup.timer
# systemctl list-timers backup.timer
NEXT                        LEFT     LAST                        PASSED  UNIT          ACTIVATES
Tue 2025-03-04 02:34:12 UTC 14h left Mon 2025-03-03 02:31:55 UTC 9h ago  backup.timer  backup.service
```

Run the job **now**, outside the schedule (good for testing):

```console
# systemctl start backup.service
$ systemctl status backup.service
$ journalctl -u backup.service -n 50
```

### Interval after boot

```ini
[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
```

Starts 5 minutes after boot and again every hour after each run began.

### Handy commands

| Command | Purpose |
|---------|---------|
| `systemctl list-timers --all` | All timers, next and last run |
| `systemctl status NAME.timer` | Timer status |
| `systemctl cat NAME.timer NAME.service` | Show unit files |
| `systemctl edit NAME.timer` | Override the schedule (see [Unit Files and Drop-ins](../06-boot-grub-systemd/05-unit-files-overrides-dropins.md)) |
| `journalctl -u NAME.service` | Job output and result |
| `systemctl list-units --failed` | Failed jobs |
| `systemd-run --on-calendar="02:00" /usr/local/bin/job.sh` | Create a transient timer |
| `systemd-run --on-active=30m /usr/local/bin/job.sh` | Run once in 30 minutes |

### cron vs. systemd timers

| | cron | systemd timer |
|---|------|---------------|
| Setup | One line | Two unit files |
| Logging | Minimal. Output is mailed or lost | **Journal**, with exit status |
| Missed jobs after downtime | No (needs anacron) | `Persistent=true` |
| Environment | Minimal, easy to get wrong | Defined in the unit, with `Environment=`, `User=` |
| Resource limits, sandboxing | Not built in | cgroups, `Nice=`, `MemoryMax=`, hardening options |
| Random delay | No | `RandomizedDelaySec=` |
| Overlap prevention | `flock` needed | A service is not started again while it runs |
| Dependencies | None | `After=`, `Wants=` |
| Portability | Works everywhere | systemd systems only |

Use whichever fits. Many teams keep cron for simple jobs and timers for important or complex ones.

## Troubleshooting scheduled jobs

### Step-by-step checklist

1. **Is the scheduler running?**

```console
$ systemctl is-active cron crond atd        # service name varies: cron (Debian), crond (RHEL)
$ systemctl list-timers --all
```

2. **Is the job defined correctly?**

```console
$ crontab -l                                  # for the right user
$ cat /etc/cron.d/*; cat /etc/crontab
$ systemd-analyze verify /etc/systemd/system/backup.timer
$ systemd-analyze calendar "..."              # the schedule evaluates to what you expect?
```

3. **Did it start?** Check the logs.

```console
$ journalctl -u cron --since today | grep myjob              # Debian-based
$ grep myjob /var/log/cron                                    # RHEL-based
$ journalctl -u backup.service --since yesterday
```

4. **Did it fail while running?** Reproduce with the **same user and environment**.

```console
$ sudo -u backup env -i PATH=/usr/bin:/bin /usr/local/bin/backup.sh
$ echo $?
```

### Typical causes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Job never appears in the log | Scheduler stopped; syntax error; wrong file name or permissions in `/etc/cron.d`; user not allowed (`cron.allow/deny`) | Check service, fix the file, check logs for "bad minute" or "bad command" |
| Starts but does nothing | Command not found (`PATH`), relative paths, missing `cd` | Use full paths; set `PATH`; use `cd /dir &&` |
| Works by hand, not in cron | Environment differences (variables, `~/.bashrc` not loaded) | Source needed files in the script; set variables in crontab |
| Job cut short | Unescaped `%` | Use `\%` |
| "Permission denied" | Script not executable; wrong user; directory permissions | `chmod +x`, `namei -l path`, check `User=` |
| SELinux or AppArmor block | Security policy | `ausearch -m avc -ts recent`, `journalctl -t audit`, fix labels |
| No output or mail | Output discarded; no mail system | Log to a file or `logger`; set `MAILTO` |
| Ran at wrong time | Time zone; daylight saving | `timedatectl`; use UTC |
| Ran twice or overlapped | Previous run not finished; two schedulers; duplicated entries | `flock`; search all crontabs and timers |
| Missing after reboot or downtime | Job missed while powered off | `Persistent=true`, anacron |
| Skipped file in `cron.daily` | File name with a dot, or not executable | `run-parts --test`; rename |
| Timer shows `n/a` for NEXT | Timer not enabled or started; service-only enabled | `systemctl enable --now NAME.timer` |
| Service `failed` after timer start | Script error or exit code non-zero | `journalctl -u NAME.service`; fix script |
| `Persistent` run happens at boot unexpectedly | Missed schedule is caught up | Expected behavior. Add `RandomizedDelaySec` or `After=` dependencies |

### Make jobs observable

A reliable scheduled job should:

- **Log** start, end, and result (with timestamp).
- Use **exit codes** correctly (non-zero on failure).
- Use a **lock** to avoid overlap.
- Send a **notification** or ping a monitoring service on success and on failure (for example, a dead-man's-switch check).
- Be **idempotent**: safe to run twice.

```bash
#!/bin/bash
set -euo pipefail
exec 9>/var/lock/backup.lock
flock -n 9 || { logger -t backup "already running"; exit 0; }
logger -t backup "start"
rsync -a /data/ /backup/data/
logger -t backup "finished OK"
```

## Common pitfalls

- **Enabling the `.service` instead of the `.timer`.** The job then runs once at boot, not on the schedule.
- **Forgetting `daemon-reload`** after editing unit files.
- **`Type=oneshot` missing** for run-to-completion jobs, so dependent units start too early.
- **Not testing the service on its own** (`systemctl start NAME.service`) before relying on the timer.
- **`OnCalendar` expression errors.** Always check with `systemd-analyze calendar`.
- **Ignoring failures.** Add alerting on `systemctl --failed` or on missing success pings.
- **Both cron and timer doing the same job** (for example after migrating). Remove the old one.
- **Security.** Run jobs as dedicated low-privilege users. Add sandboxing options (`ProtectSystem=`, `NoNewPrivileges=`).

## Further reading

- [`systemd.timer(5)`](https://man7.org/linux/man-pages/man5/systemd.timer.5.html)
- [`systemd.time(7)`](https://man7.org/linux/man-pages/man7/systemd.time.7.html)
- [`systemd-analyze(1)`](https://man7.org/linux/man-pages/man1/systemd-analyze.1.html)
- [`flock(1)`](https://man7.org/linux/man-pages/man1/flock.1.html)
