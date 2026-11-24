# Service Management with systemctl

## Overview

`systemctl` is the command you use every day to start, stop, and inspect services. Commands that change state need **root** (or `sudo`).

## Commands

### Control a service

| Command | Purpose |
|---------|---------|
| `systemctl start NAME` | Start now |
| `systemctl stop NAME` | Stop now |
| `systemctl restart NAME` | Stop and start (connections drop) |
| `systemctl reload NAME` | Ask the service to re-read its configuration (no restart) |
| `systemctl reload-or-restart NAME` | Reload if supported, otherwise restart |
| `systemctl status NAME` | Show state and recent log lines |
| `systemctl enable NAME` | Start automatically at boot |
| `systemctl disable NAME` | Do not start at boot |
| `systemctl enable --now NAME` | Enable **and** start now |
| `systemctl disable --now NAME` | Disable **and** stop now |
| `systemctl mask NAME` | Make it **impossible** to start (links to `/dev/null`) |
| `systemctl unmask NAME` | Remove the mask |

`NAME` can omit `.service`: `systemctl restart nginx`.

### Check state

```console
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2025-03-03 09:01:12 UTC; 2h 10min ago
       Docs: man:nginx(8)
    Process: 701 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
   Main PID: 712 (nginx)
      Tasks: 3 (limit: 4557)
     Memory: 5.2M
        CPU: 120ms
     CGroup: /system.slice/nginx.service
             ├─712 "nginx: master process /usr/sbin/nginx"
             └─713 "nginx: worker process"

Mar 03 09:01:12 server01 systemd[1]: Started nginx.service.
```

| Line | Meaning |
|------|---------|
| `Loaded:` | Unit file path and whether it is `enabled`, `disabled`, `static`, or `masked` |
| `Active:` | `active (running)`, `inactive (dead)`, `failed`, `activating` |
| `Main PID` | Process ID of the main process |
| Last lines | Recent log messages from the journal |

Script-friendly checks (they print a word and set the exit code):

```console
$ systemctl is-active nginx
active
$ systemctl is-enabled nginx
enabled
$ systemctl is-failed nginx
$ systemctl is-active --quiet nginx && echo running
```

### List units

| Command | Purpose |
|---------|---------|
| `systemctl list-units` | Loaded and active units |
| `systemctl list-units --type=service --state=running` | Running services |
| `systemctl --failed` | Failed units |
| `systemctl list-unit-files --type=service` | All installed services and their enable state |
| `systemctl list-timers` | Timers |
| `systemctl list-sockets` | Sockets |

```console
$ systemctl --failed
  UNIT               LOAD   ACTIVE SUB    DESCRIPTION
● myapp.service      loaded failed failed My application
```

### Inspect a unit

```console
$ systemctl cat nginx                   # show the unit file and drop-ins
$ systemctl show nginx -p MainPID,ActiveState,MemoryCurrent
$ systemctl list-dependencies nginx
$ systemctl show -p FragmentPath,DropInPaths nginx
```

### After changing unit files

```console
# systemctl daemon-reload               # re-read unit files (required after any change)
# systemctl restart myapp
# systemctl reset-failed myapp          # clear the "failed" state and restart counters
```

### System state commands

| Command | Purpose |
|---------|---------|
| `systemctl reboot` | Reboot |
| `systemctl poweroff` | Shut down |
| `systemctl suspend` | Suspend |
| `systemctl rescue` | Switch to rescue mode |
| `systemctl is-system-running` | `running`, `degraded` (some unit failed), `starting` |

> [!WARNING]
> `systemctl reboot`, `poweroff`, and `isolate` act immediately. On a remote server, double-check you are on the right host.

### Enabled vs. active

| | Meaning |
|---|---------|
| **active** | Running **now** |
| **enabled** | Will start at the **next boot** |

A service can be active but disabled (will not return after reboot), or enabled but stopped. The `enable --now` form sets both.

## Common pitfalls

- **Enabling without starting** (or the reverse). Use `enable --now`.
- **Forgetting `daemon-reload`** after editing a unit file. systemd warns: "unit file changed on disk".
- **Service "failed" with "start-limit-hit".** It restarted too often. Fix the cause, then `systemctl reset-failed NAME`.
- **`restart` where `reload` is enough** drops connections. Use `reload` for configuration changes if supported.
- **Service name differences by distribution:** `sshd` on RHEL-based and `ssh` on Debian-based (with `sshd` as an alias on newer releases); `httpd` vs. `apache2`; `crond` vs. `cron`.
- **Masked units** will refuse to start with "Unit is masked". Run `systemctl unmask`.
- **`status` truncates lines.** Use `systemctl status NAME -l --no-pager` or `journalctl -u NAME`.

## Further reading

- [`systemctl(1)`](https://man7.org/linux/man-pages/man1/systemctl.1.html)
- [`systemd.service(5)`](https://man7.org/linux/man-pages/man5/systemd.service.5.html)
