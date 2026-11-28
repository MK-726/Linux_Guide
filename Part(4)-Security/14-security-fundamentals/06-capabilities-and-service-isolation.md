# Linux Capabilities and Service Isolation

## Overview

If a service is compromised, the attacker gets that service's privileges. **Isolation** limits what those privileges allow. This page deepens the introduction to capabilities ([Linux Capabilities (Introduction)](../../part-1-foundations/04-users-groups-and-permissions/08-capabilities-introduction.md)) and shows how to **sandbox services** with systemd, namespaces, and containers.

## Key concepts

### Capabilities in depth

Capabilities divide root's power. Each process has several **sets**:

| Set | Meaning |
|-----|---------|
| **Permitted** | The maximum a process may use |
| **Effective** | What the kernel checks right now |
| **Inheritable** | Allowed to pass to children through `exec` (with file capabilities) |
| **Bounding** | A hard limit: capabilities outside it can never be gained |
| **Ambient** | Preserved across `exec` for unprivileged programs (the practical way to give a service one capability) |

Capabilities to treat as dangerous (near-root):

| Capability | Why dangerous |
|------------|---------------|
| `CAP_SYS_ADMIN` | A huge catch-all: mounting, namespaces, many device operations |
| `CAP_DAC_OVERRIDE`, `CAP_DAC_READ_SEARCH` | Ignore file permissions |
| `CAP_SETUID`, `CAP_SETGID` | Become any user |
| `CAP_SYS_PTRACE` | Inspect and modify other processes |
| `CAP_SYS_MODULE` | Load kernel code |
| `CAP_NET_ADMIN` | Change network configuration, firewall |
| `CAP_SYS_RAWIO` | Raw device access |
| `CAP_CHOWN`, `CAP_FOWNER` | Change ownership and permissions of any file |

### Inspect capabilities

```console
$ grep Cap /proc/self/status
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000
$ capsh --decode=00000000a80425fb              # turn the hex mask into names
$ getpcaps $(pgrep -o nginx)
$ capsh --print                                 # for the current shell
# getcap -r / 2>/dev/null                       # files with capabilities (audit these)
# setcap -r /path/to/file                       # remove file capabilities
```

### Drop capabilities for a single command

```console
# capsh --drop=cap_net_raw,cap_sys_admin -- -c 'whoami; grep Cap /proc/self/status'
# setpriv --bounding-set=-all,+net_bind_service --reuid=1000 --regid=1000 --clear-groups --inh-caps=-all /usr/bin/myserver
```

## Key concepts: sandboxing services with systemd

systemd can confine a service with **a few lines in the unit file**. These options use namespaces, capabilities, seccomp, and mount tricks behind the scenes.

| Directive | Effect |
|-----------|--------|
| `User=`, `Group=` | Run as a dedicated unprivileged account |
| `DynamicUser=yes` | Create a temporary user for the service automatically |
| `NoNewPrivileges=yes` | The process and its children can never gain privileges (blocks SUID, file capabilities) |
| `CapabilityBoundingSet=` | Limit the capabilities available (empty = none) |
| `AmbientCapabilities=` | Give specific capabilities to a non-root service |
| `ProtectSystem=strict` | Mount the **entire filesystem read-only** (except `/dev`, `/proc`, `/sys`). `full` protects `/usr`, `/boot`, `/etc`. `true` protects `/usr` and `/boot` |
| `ReadWritePaths=` | Paths that remain writable |
| `StateDirectory=`, `LogsDirectory=`, `CacheDirectory=`, `RuntimeDirectory=` | Create and own writable directories for the service automatically |
| `ProtectHome=yes` | `/home`, `/root`, `/run/user` are inaccessible. `read-only` makes them read-only |
| `PrivateTmp=yes` | Private `/tmp` and `/var/tmp` |
| `PrivateDevices=yes` | Minimal `/dev` (no physical devices) |
| `PrivateNetwork=yes` | No network access except loopback |
| `PrivateUsers=yes` | User namespace |
| `ProtectKernelTunables=yes` | `/proc/sys`, `/sys` read-only |
| `ProtectKernelModules=yes` | Cannot load modules |
| `ProtectKernelLogs=yes` | No access to the kernel log |
| `ProtectControlGroups=yes` | cgroups read-only |
| `ProtectClock=yes`, `ProtectHostname=yes` | Cannot change the clock or hostname |
| `ProtectProc=invisible` | Hide other users' processes |
| `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX` | Allowed socket types |
| `RestrictNamespaces=yes` | Cannot create namespaces |
| `RestrictRealtime=yes` | No real-time scheduling |
| `RestrictSUIDSGID=yes` | Cannot create SUID/SGID files |
| `MemoryDenyWriteExecute=yes` | No memory that is both writable and executable (breaks JIT engines such as Java and Node, so test) |
| `LockPersonality=yes` | Locks the execution domain |
| `SystemCallFilter=@system-service` | Allow only a standard set of syscalls (seccomp). Add `~@privileged @resources` to remove more |
| `SystemCallArchitectures=native` | Native syscalls only |
| `IPAddressDeny=any` / `IPAddressAllow=` | Per-service IP filtering |
| `InaccessiblePaths=`, `ReadOnlyPaths=` | Hide or protect specific paths |
| `UMask=0027` | Permissions of created files |

### Measure the exposure

```console
$ systemd-analyze security nginx.service
  NAME                                  DESCRIPTION                              EXPOSURE
✗ PrivateNetwork=                       Service has access to the host's network      0.5
✗ User=/DynamicUser=                    Service runs as root user                      0.4
✓ NoNewPrivileges=                      Service processes cannot acquire new privileges
...
→ Overall exposure level for nginx.service: 8.4 EXPOSED 🙁
$ systemd-analyze security --no-pager | sort -k2 | head      # all services
```

A lower **exposure level** is better (0 = well-sandboxed, 10 = UNSAFE). Scores are heuristics. Not every directive fits every service. Use the list of `✗` items as ideas.

### Example: sandboxed application service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
User=myapp
Group=myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/app.conf
Restart=on-failure

# Filesystem
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
StateDirectory=myapp
LogsDirectory=myapp
ConfigurationDirectory=myapp
ReadWritePaths=/var/lib/myapp

# Privileges
NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=
RestrictSUIDSGID=yes
LockPersonality=yes

# Kernel and system
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
MemoryDenyWriteExecute=yes

[Install]
WantedBy=multi-user.target
```

Apply and test carefully:

```console
# systemctl daemon-reload && systemctl restart myapp
# systemctl status myapp
# journalctl -u myapp -n 50                         # permission errors point to a directive that is too strict
$ systemd-analyze security myapp.service
```

If the service fails after adding a directive, read the journal, then loosen **one** directive at a time. Syscall denials show up in the audit log (`ausearch -m seccomp`) or as `Bad system call` (SIGSYS).

### Other isolation techniques

| Technique | Strength | Notes |
|-----------|----------|-------|
| Separate **user account** | Basic | The foundation. One account per service |
| **chroot** | Weak | Changes the visible root directory. **Not a security boundary** for root processes |
| **Namespaces** (`unshare`, `bwrap`) | Medium | Limit visibility of processes, network, mounts, users |
| **systemd sandboxing** | Medium | Easy and well maintained (above) |
| **seccomp** | Medium | Filters syscalls. Used by containers, browsers, systemd |
| **Landlock** | Medium | Self-restriction of filesystem access (needs program support) |
| **MAC (SELinux, AppArmor)** | Strong | Policy-enforced confinement, independent of the user ID (Chapter 16) |
| **Containers** (Podman, Docker) | Medium | Namespaces + cgroups + capabilities + seccomp + MAC. **Share the host kernel** |
| **Rootless containers** | Better | The container's root maps to an unprivileged host user |
| **gVisor, Kata Containers** | Stronger | User-space kernel or lightweight VM per container |
| **Virtual machines** | Strong | Separate kernel. A hypervisor boundary |
| **Separate hosts / networks** | Strongest | Physical or network isolation |

Choose isolation based on the **value of the asset** and the **risk of the software**. Untrusted code (customer uploads, third-party plugins) deserves a stronger boundary.

### Container hardening basics

```console
$ podman run --rm --read-only --cap-drop=ALL --security-opt=no-new-privileges \
      --user 1000:1000 --pids-limit 200 --memory 512m -p 127.0.0.1:8080:8080 myimage
```

| Flag | Effect |
|------|--------|
| `--read-only` | Read-only root filesystem (add `--tmpfs /tmp` where needed) |
| `--cap-drop=ALL --cap-add=...` | Remove all capabilities and add only what is needed |
| `--security-opt=no-new-privileges` | Block privilege gain |
| `--user` | Non-root user inside the container |
| `--pids-limit`, `--memory`, `--cpus` | Resource limits |
| `-p 127.0.0.1:8080:8080` | Publish only to the loopback |

> [!WARNING]
> Never use `--privileged`, mount the Docker socket (`/var/run/docker.sock`) into a container, share the host PID/network namespaces, or add `SYS_ADMIN` unless you fully accept that this equals root on the host. Membership in the `docker` group is equivalent to root.

## Common pitfalls

- **Giving a service `CAP_SYS_ADMIN`** to fix one permission error. Find the specific capability instead.
- **Running services as root** because it "works".
- **Using chroot as a sandbox.** It protects against careless mistakes, not determined attackers.
- **Adding all sandbox options at once.** The service breaks and you cannot tell which option caused it. Add in steps.
- **`MemoryDenyWriteExecute=yes` on JIT runtimes** (Java, Node.js, .NET, PHP with JIT, Python with some libraries) causes crashes.
- **`ProtectSystem=strict` without `ReadWritePaths=` / `StateDirectory=`**, so the service cannot write its data.
- **`NoNewPrivileges` and SUID helpers.** Services using `sudo` or SUID programs stop working. Rework the design.
- **Trusting a container as a VM.** Kernel exploits and misconfigurations (privileged mode, host mounts) break out of containers.
- **Capabilities lost after package updates** (`setcap` on files). Prefer systemd `AmbientCapabilities=`.
- **Ignoring the score.** `systemd-analyze security` shows quick wins. Review default and third-party units regularly.

## Further reading

- [`systemd.exec(5)`: sandboxing](https://man7.org/linux/man-pages/man5/systemd.exec.5.html)
- [`capabilities(7)`](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [`systemd-analyze(1)`: security](https://man7.org/linux/man-pages/man1/systemd-analyze.1.html)
- [Landlock documentation](https://docs.kernel.org/userspace-api/landlock.html)
- [Podman security guidance](https://docs.podman.io/en/latest/markdown/podman-run.1.html)
- [NIST SP 800-190: Application Container Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final)
