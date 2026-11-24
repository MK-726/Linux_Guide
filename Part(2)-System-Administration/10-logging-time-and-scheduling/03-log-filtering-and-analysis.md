# Log Filtering and Analysis

## Overview

Logs contain thousands of lines. The skill is to narrow down quickly: by **time**, **service**, **severity**, and **pattern**. This page shows practical filters with `journalctl`, `grep`, `awk`, and related tools.

## Commands

### Start narrow: time, unit, severity

```console
$ journalctl -u nginx --since "2025-03-03 09:00" --until "09:30" -p warning
$ journalctl -b -p err                       # errors since boot
$ journalctl -k --since "1 hour ago"         # kernel messages
$ journalctl -g "timeout|refused" -u myapp   # regular expression on the message
```

See [journalctl](../06-boot-grub-systemd/06-journalctl.md) for all filters.

### Text log tools

| Task | Command |
|------|---------|
| Follow a log | `tail -f /var/log/syslog` or `tail -F` (follows rotation) |
| Follow several logs | `tail -f /var/log/nginx/access.log /var/log/nginx/error.log` |
| Search with context | `grep -C 3 "error" app.log` |
| Case-insensitive, count | `grep -ic "failed" auth.log` |
| Compressed rotated logs | `zgrep "ERROR" /var/log/app/app.log*.gz` |
| Search several files and show file names | `grep -rn "denied" /var/log/` |
| Lines between two times (same day) | `awk '$3 >= "10:00:00" && $3 <= "10:30:00"' /var/log/syslog` |
| Exclude noise | `grep -v -E "healthcheck\|GET /ping" access.log` |
| Pager with search and follow | `less +F file` (Ctrl+C to stop following, `F` to resume) |

### Patterns that answer common questions

**Who failed to log in via SSH?** (Debian-based: `auth.log`, RHEL-based: `secure`)

```console
$ grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
    842 203.0.113.9
    120 198.51.100.4
$ journalctl -u ssh --since today | grep -c "Failed password"
```

**Who logged in, when, and from where?**

```console
$ last -n 10                     # successful logins (/var/log/wtmp)
$ lastb -n 10                    # failed logins (/var/log/btmp; needs root)
$ lastlog | grep -v "Never"      # last login of each account
$ who -a ; w
```

**Which sudo commands ran?**

```console
$ journalctl _COMM=sudo --since today
$ grep "COMMAND=" /var/log/auth.log | tail
```

**Web server analysis (combined log format):**

```console
$ awk '{print $9}' access.log | sort | uniq -c | sort -rn           # status codes
$ awk '$9 >= 500 {print $7}' access.log | sort | uniq -c | sort -rn | head    # URLs with errors
$ awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5           # top client IPs
$ awk '{print substr($4,2,17)}' access.log | uniq -c                          # requests per minute
$ cut -d'"' -f6 access.log | sort | uniq -c | sort -rn | head                 # user agents
```

**Errors per minute (find spikes):**

```console
$ grep -i error app.log | cut -c1-16 | uniq -c
```

**OOM kills, disk errors, service crashes:**

```console
$ dmesg -T | grep -i -E 'out of memory|killed process|i/o error|segfault'
$ journalctl -p err -b | grep -i -E 'failed|error' | head
$ journalctl --since yesterday -u myapp | grep -E 'Main process exited|Failed with result'
```

### JSON logs

```console
$ jq -r 'select(.level=="error") | [.time,.msg] | @tsv' app.json.log      # jq is the standard tool
$ journalctl -u myapp -o json | jq -r '.MESSAGE'
```

### Helpful tools

| Tool | Purpose |
|------|---------|
| `lnav` | Interactive log navigator with SQL queries, auto-detects formats and merges logs by time |
| `goaccess` | Real-time web log analyzer and reports |
| `multitail` | View several logs in split windows |
| `ccze` | Colorize logs |
| `jq` | JSON processing |
| `fail2ban` | Ban IP addresses after repeated failures |
| `logwatch` | Daily summary reports by mail |
| Central platforms | ELK/OpenSearch, Grafana Loki, Graylog, Splunk (Chapter 25) |

### A repeatable approach

1. **When** did it happen? Set a time window with `--since` and `--until`. Check the **time zone**.
2. **What** is affected? Filter by unit, program, or host.
3. **Start from errors**, then widen the window to see what happened **before** the first error.
4. **Correlate** across components: application, web server, database, kernel, network, authentication.
5. **Count** and **group** (`sort | uniq -c`) to see patterns versus one-off events.
6. **Confirm** the theory from a second source (metrics, another log).
7. **Save the evidence** (copy relevant lines to a ticket) before logs rotate.

## Common pitfalls

- **Wrong time zone or unsynchronized clocks.** Logs from different hosts do not line up. Use UTC for correlation (`journalctl --utc`).
- **Reading only the last lines.** The cause often appears **before** the visible errors.
- **`tail -f` stops after rotation.** Use `tail -F`.
- **Missing rotated logs.** Older entries are in `.1`, `.2.gz`. Use `zgrep`.
- **Parsing logs with `awk` fields that shift** because of variable-length messages. Test on a small sample.
- **Grepping for the wrong text** (case, spelling). Try `-i` and shorter patterns.
- **Large files in `less` or editors.** Use `grep`, `tail`, or `less +G` with search, not a full editor.
- **Exposing sensitive logs.** Logs contain IP addresses, usernames, sometimes tokens. Share carefully.
- **Taking noise for the cause.** Some "errors" are always present. Compare with a normal day.

## Further reading

- [`journalctl(1)`](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [lnav](https://lnav.org)
- [`jq` manual](https://jqlang.github.io/jq/manual/)
- [fail2ban](https://github.com/fail2ban/fail2ban)
