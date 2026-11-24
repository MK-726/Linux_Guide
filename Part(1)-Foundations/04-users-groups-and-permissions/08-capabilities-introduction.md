# Linux Capabilities (Introduction)

## Overview

Traditionally, a process was either root (can do everything) or not (very limited). **Capabilities** split root's power into about 40 small pieces. A program can get only the one it needs, for example, "bind to port 80" without being full root. This follows the principle of **least privilege**.

## Key concepts

### Common capabilities

| Capability | Allows |
|------------|--------|
| `CAP_NET_BIND_SERVICE` | Bind to TCP/UDP ports below 1024 |
| `CAP_NET_RAW` | Use raw sockets (for example `ping`) |
| `CAP_NET_ADMIN` | Change network configuration |
| `CAP_CHOWN` | Change file owners |
| `CAP_DAC_OVERRIDE` | Ignore file read/write/execute permission checks |
| `CAP_KILL` | Send signals to any process |
| `CAP_SETUID` / `CAP_SETGID` | Change process user and group IDs |
| `CAP_SYS_ADMIN` | A very broad set of administrative operations (treat as "almost root") |
| `CAP_SYS_PTRACE` | Trace and inspect other processes |
| `CAP_SYS_TIME` | Change the system clock |

> [!WARNING]
> `CAP_SYS_ADMIN` and `CAP_DAC_OVERRIDE` give powers close to full root. Avoid granting them.

### Capability sets of a process

| Set | Meaning |
|-----|---------|
| **Permitted** | Capabilities the process may use |
| **Effective** | Capabilities currently in use (checked by the kernel) |
| **Inheritable** | Capabilities passed through `exec` (with file capabilities) |
| **Bounding** | A limit: capabilities that can never be gained |
| **Ambient** | Capabilities kept across `exec` for non-privileged programs |

Beginners can think of it as: *Permitted = what I may have*, *Effective = what I am using now*, *Bounding = the hard limit*.

### File capabilities

A capability can be attached to an executable file, similar to SUID but much narrower. In `setcap`, the letters mean: `e` effective, `p` permitted, `i` inheritable.

## Commands

### View capabilities

```console
$ capsh --print                  # your shell's capabilities
Current: =
Bounding set =cap_chown,cap_dac_override,...

$ getpcaps $$                    # capabilities of a PID
$ grep Cap /proc/self/status
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000

$ capsh --decode=000001ffffffffff
```

For a process started as root, `CapEff` has many bits set. For a normal user, it is `0`.

### Files with capabilities

```console
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep

$ getcap -r /usr 2>/dev/null     # find all files with capabilities
```

**Requires root:**

```console
# setcap 'cap_net_bind_service=+ep' /usr/local/bin/myserver     # allow port 80/443 without root
# getcap /usr/local/bin/myserver
/usr/local/bin/myserver cap_net_bind_service=ep
# setcap -r /usr/local/bin/myserver                              # remove
```

> [!NOTE]
> Capabilities are lost when the file is replaced (for example after an upgrade or `cp`). The filesystem must support extended attributes. File capabilities do not work on interpreted scripts, only on compiled programs.

### systemd services

systemd can restrict or grant capabilities per service. This is the preferred approach for services:

```ini
[Service]
User=myapp
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes
```

| Directive | Effect |
|-----------|--------|
| `AmbientCapabilities=` | Give the service these capabilities even though it runs as a normal user |
| `CapabilityBoundingSet=` | Limit the service to only these |
| `NoNewPrivileges=yes` | Block SUID and capability gain |

This runs the service as a non-root user that can still bind to low ports.

### Containers

Container runtimes (Docker and Podman) drop most capabilities by default. Add only what is needed: `docker run --cap-drop ALL --cap-add NET_BIND_SERVICE ...`. See Chapter 22.

## Common pitfalls

- **Using `setcap` on scripts.** It has no effect on interpreted files.
- **Granting broad capabilities** such as `CAP_SYS_ADMIN` to avoid troubleshooting. Find the specific capability needed.
- **Forgetting that capabilities disappear when the binary is updated.** Re-apply after upgrades, or prefer systemd `AmbientCapabilities=`.
- **Forgetting to audit.** Run `getcap -r / 2>/dev/null` regularly, along with the SUID search.
- **Confusing capabilities with permissions.** Capabilities do not replace file permissions. They add specific kernel-level rights.

Chapter 14 returns to capabilities as part of host hardening.

## Further reading

- [`capabilities(7)`](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [`setcap(8)`](https://man7.org/linux/man-pages/man8/setcap.8.html)
- [`systemd.exec(5)`: capabilities](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)
