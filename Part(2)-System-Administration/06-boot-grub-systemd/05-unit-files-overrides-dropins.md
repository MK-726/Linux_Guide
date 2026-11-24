# Unit Files, Overrides, and Drop-ins

## Overview

A **unit file** tells systemd how to run a service. You write your own for custom applications, and you **override** vendor units to change their behavior without editing package files.

## Key concepts

### Anatomy of a service unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
Documentation=https://example.com/docs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=-/etc/myapp/myapp.env
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/app.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

| Section | Purpose |
|---------|---------|
| `[Unit]` | Description, dependencies, ordering |
| `[Service]` | How to run the process |
| `[Install]` | What `systemctl enable` does (which target pulls it in) |

### Important [Service] settings

| Directive | Meaning |
|-----------|---------|
| `Type=simple` | The process in `ExecStart` **is** the service (default) |
| `Type=forking` | The process forks and the parent exits (old-style daemons). Use with `PIDFile=` |
| `Type=oneshot` | Runs a task and exits. Often with `RemainAfterExit=yes` |
| `Type=notify` | The service tells systemd when it is ready (`sd_notify`) |
| `ExecStart=` | Command to start. **Use full paths** |
| `ExecStartPre=` / `ExecStartPost=` | Commands before and after start |
| `ExecStop=` / `ExecReload=` | Custom stop and reload commands |
| `Restart=` | `no`, `on-failure`, `always`, `on-abnormal` |
| `RestartSec=` | Wait before restarting |
| `User=`, `Group=` | Run as this account |
| `WorkingDirectory=` | Start directory |
| `Environment=KEY=value` | Set a variable |
| `EnvironmentFile=` | Read variables from a file (prefix `-` to ignore a missing file) |
| `TimeoutStartSec=`, `TimeoutStopSec=` | How long systemd waits |
| `LimitNOFILE=` | Open-file limit |
| `StandardOutput=`, `StandardError=` | Where output goes (journal by default) |

> [!NOTE]
> systemd does **not** run `ExecStart` through a shell. Pipes, `&&`, redirects, and `$VAR` expansion from the shell do not work. To use them, call `/bin/sh -c '...'`. Use `${VAR}` for systemd's own variable substitution.

### Hardening options

systemd can reduce a service's access with a few lines. These improve security a lot:

```ini
[Service]
NoNewPrivileges=yes
ProtectSystem=strict          # filesystem read-only except listed paths
ProtectHome=yes               # no access to /home
PrivateTmp=yes                # private /tmp
ReadWritePaths=/var/lib/myapp
CapabilityBoundingSet=
```

Check the result with `systemd-analyze security myapp.service`.

### Overrides and drop-ins

Never edit files under `/usr/lib/systemd/system/`. Instead, create a **drop-in** that changes only what you need. It survives package upgrades.

```console
# systemctl edit nginx
```

This opens an editor on `/etc/systemd/system/nginx.service.d/override.conf`. Enter only the settings you want to change:

```ini
[Service]
LimitNOFILE=65536
Restart=always
```

Save and exit. systemd runs `daemon-reload` for you. Apply with `systemctl restart nginx`.

| Command | Purpose |
|---------|---------|
| `systemctl edit NAME` | Create or edit a drop-in (`override.conf`) |
| `systemctl edit --full NAME` | Copy the whole unit to `/etc/systemd/system/` and edit it |
| `systemctl cat NAME` | Show the unit plus every drop-in |
| `systemctl revert NAME` | Remove your overrides and return to the vendor version |
| `systemd-analyze verify FILE` | Check a unit file for errors |

**Resetting list-type settings:** Some settings (like `ExecStart=`) can appear more than once. In a drop-in, clear the old value first with an empty assignment:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/nginx -g 'daemon off;' -c /etc/nginx/custom.conf
```

Without the empty `ExecStart=` line, systemd reports "more than one ExecStart".

### Template units

A unit with `@` in its name is a template. The text after `@` is the **instance** and is available as `%i`.

```ini
# /etc/systemd/system/worker@.service
[Service]
ExecStart=/usr/local/bin/worker --id %i
```

```console
# systemctl start worker@1 worker@2
```

## Commands: create a service step by step

```console
# useradd -r -s /usr/sbin/nologin myapp
# vim /etc/systemd/system/myapp.service           # write the unit
# systemd-analyze verify /etc/systemd/system/myapp.service
# systemctl daemon-reload
# systemctl enable --now myapp
# systemctl status myapp
# journalctl -u myapp -f
```

## Common pitfalls

- **Relative paths in `ExecStart`.** Always use the full path to the program.
- **Expecting shell features.** Redirection and pipes need `/bin/sh -c`.
- **Forgetting `daemon-reload`** after changing a file in `/etc/systemd/system/`.
- **Wrong `Type=`.** A forking daemon with `Type=simple` appears to exit immediately and gets marked failed. A foreground program with `Type=forking` times out.
- **Service starts too early.** Add `After=network-online.target` and `Wants=network-online.target` when the application needs the network configured.
- **Service restarts forever.** Use `StartLimitBurst=` and `StartLimitIntervalSec=` in `[Unit]`, and fix the root cause.
- **File permission errors under `ProtectSystem`.** Grant paths with `ReadWritePaths=` or `StateDirectory=`.
- **Environment files with spaces or quotes** behave differently than shell scripts. Use simple `KEY=value` lines.

## Further reading

- [`systemd.service(5)`](https://man7.org/linux/man-pages/man5/systemd.service.5.html)
- [`systemd.exec(5)`](https://man7.org/linux/man-pages/man5/systemd.exec.5.html)
- [`systemd.unit(5)`](https://man7.org/linux/man-pages/man5/systemd.unit.5.html)
- [`systemd-analyze(1)`](https://man7.org/linux/man-pages/man1/systemd-analyze.1.html)
