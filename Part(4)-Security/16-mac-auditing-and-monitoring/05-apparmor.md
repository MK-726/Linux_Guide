# AppArmor

## Overview

**AppArmor** is the MAC system on Ubuntu, Debian (available), and SUSE (SLES 15 and openSUSE). It confines programs with **profiles** that list the files, capabilities, and network access a program may use. Profiles are **path-based** and plain text, so they are easier to read and write than SELinux policy.

## Key concepts

### How it works

- A **profile** is attached to an **executable path** (for example `/usr/sbin/nginx`).
- When that program runs, the kernel confines it according to its profile.
- Anything not allowed by the profile is **denied** (in enforce mode).
- Programs **without** a profile are **unconfined**.
- Profiles are loaded into the kernel from `/etc/apparmor.d/`.

### Modes

| Mode | Behavior |
|------|----------|
| **Enforce** | Violations are blocked and logged |
| **Complain** | Violations are **allowed but logged** (for developing profiles) |
| **Unconfined** | No profile. No restrictions |
| **Kill** | Violations kill the process (rare) |

### Files and directories

| Path | Purpose |
|------|---------|
| `/etc/apparmor.d/` | Profiles. Names reflect the path: `usr.sbin.nginx` for `/usr/sbin/nginx` |
| `/etc/apparmor.d/abstractions/` | Reusable rule sets (`base`, `nameservice`, `ssl_certs`, `apache2-common`) |
| `/etc/apparmor.d/tunables/` | Variables (`@{HOME}`, `@{PROC}`) |
| `/etc/apparmor.d/local/` | **Your local additions** to packaged profiles. Included by the profile and preserved on updates |
| `/etc/apparmor.d/disable/` | Links that disable profiles |
| `/sys/kernel/security/apparmor/profiles` | Loaded profiles |

### Profile syntax

```
#include <tunables/global>

/usr/local/bin/myapp {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,
  network inet stream,
  network inet6 stream,

  /usr/local/bin/myapp         mr,
  /etc/myapp/**                r,
  /var/lib/myapp/**            rwk,
  /var/log/myapp/*.log         w,
  /run/myapp.pid               rw,
  owner /tmp/myapp-*           rw,

  deny /home/** rwklx,
  deny /etc/shadow rwklx,

  #include <local/usr.local.bin.myapp>
}
```

File permission letters:

| Letter | Meaning |
|--------|---------|
| `r` | Read |
| `w` | Write |
| `a` | Append only |
| `k` | File locking |
| `l` | Create hard links |
| `m` | Memory-map with execute |
| `ix` | Execute and **inherit** the current profile |
| `px` / `Px` | Execute under **another profile** (`P` cleans the environment) |
| `cx` | Execute under a **child profile** |
| `ux` / `Ux` | Execute **unconfined** (avoid) |

Globs: `*` matches within one directory level, `**` matches across directories, `?` one character, `{a,b}` alternatives. `owner` limits a rule to files owned by the process's user. `deny` rules **win over allow** and are not logged by default unless `audit deny` is used.

## Commands

Install tools: **Debian-based** `apt install apparmor apparmor-utils apparmor-profiles apparmor-profiles-extra`; **SUSE** `zypper install apparmor-utils apparmor-profiles`.

### Status

```console
# aa-status
apparmor module is loaded.
52 profiles are loaded.
50 profiles are in enforce mode.
   /usr/sbin/nginx
   /usr/bin/man
   docker-default
   snap.firefox.firefox
2 profiles are in complain mode.
   /usr/local/bin/myapp
3 processes have profiles defined.
3 processes are in enforce mode.
   /usr/sbin/nginx (1020)
0 processes are unconfined but have a profile defined.
# aa-unconfined                                  # network-facing programs WITHOUT a profile
# systemctl status apparmor
# cat /sys/module/apparmor/parameters/enabled    # Y
# ps -eZ | head                                  # AppArmor labels (profile name) on supported systems
```

### Change mode

```console
# aa-enforce /etc/apparmor.d/usr.sbin.nginx         # or: aa-enforce /usr/sbin/nginx
# aa-complain /etc/apparmor.d/usr.sbin.nginx        # log violations instead of blocking
# aa-disable /etc/apparmor.d/usr.sbin.nginx         # unload and disable (avoid)
# apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx # (re)load one profile after editing
# apparmor_parser -R /etc/apparmor.d/usr.sbin.nginx # remove from the kernel
# systemctl reload apparmor                         # reload all profiles
```

### Find denials

```console
# journalctl -k | grep -i 'apparmor="DENIED"'
# dmesg | grep -i apparmor
# grep 'apparmor="DENIED"' /var/log/audit/audit.log           # if auditd is running
# journalctl -t kernel --since "10 min ago" | grep DENIED
```

An example denial:

```
audit: type=1400 audit(1741000000.123:78): apparmor="DENIED" operation="open" profile="/usr/sbin/nginx" name="/srv/web/index.html" pid=1020 comm="nginx" requested_mask="r" denied_mask="r" fsuid=33 ouid=0
```

| Field | Meaning |
|-------|---------|
| `apparmor="DENIED"` | Blocked (`ALLOWED` appears in complain mode) |
| `operation="open"` | What was attempted (`open`, `mknod`, `exec`, `capable`, `connect`, `file_perm`) |
| `profile=` | The confining profile |
| `name=` | The path or resource |
| `requested_mask` / `denied_mask` | `r`, `w`, `x`... |

### Fix a denial for a packaged profile

Add the rule to the **local override** file (do not edit the packaged profile):

```console
# echo '/srv/web/** r,' >> /etc/apparmor.d/local/usr.sbin.nginx
# apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
# systemctl restart nginx
```

### Build a profile for your own program

```console
# aa-genprof /usr/local/bin/myapp
# (in another terminal: run the application through all its normal functions)
# (back in aa-genprof: press S to scan the logs, then answer questions: Allow, Deny, Glob, ...)
# (press F to finish: it saves the profile in enforce mode)
```

Or iterate manually:

```console
# aa-autodep /usr/local/bin/myapp               # create a basic profile (complain mode)
# systemctl restart myapp; <exercise the application>
# aa-logprof                                    # read the logs and propose rule changes interactively
# aa-enforce /etc/apparmor.d/usr.local.bin.myapp
```

Review every suggested rule. Do not grant `/**` or `rwx` broadly.

### Notifications and tools

| Tool | Purpose |
|------|---------|
| `aa-notify` | Desktop notifications of denials |
| `aa-exec -p PROFILE command` | Run a command under a profile |
| `aa-cleanprof` | Remove unneeded rules from a profile |
| `apparmor_parser -p profile` | Check/preprocess a profile |
| `apparmor_status` | Same as `aa-status` |

### Where AppArmor protects you by default

- **Ubuntu:** profiles for many system services, `cups`, `man`, `tcpdump`, and **Snap packages** (strict confinement through AppArmor plus seccomp).
- **Containers:** Docker's `docker-default` profile and LXD/Podman profiles confine containers.
- **Libvirt:** per-VM profiles (`libvirt-<uuid>`).
- **SUSE:** YaST includes an AppArmor module, and `aa-*` tools are the same.

### Debian and kernel parameters

Recent Debian enables AppArmor by default. To be sure: kernel parameters `apparmor=1 security=apparmor`, and install the `apparmor`, `apparmor-utils` packages. Check with `aa-status`.

## Common pitfalls

- **Editing packaged profiles directly.** Updates overwrite them. Use `/etc/apparmor.d/local/`.
- **Forgetting to reload** after editing (`apparmor_parser -r`).
- **Complain mode left on.** Violations are only logged. Switch back to enforce.
- **Path problems.** Rules match the path used. **Bind mounts, symlinks, and chroots** can change paths. Use real paths and test.
- **Too broad rules** (`/** rwx`) that defeat confinement.
- **`ux` or `Ux`** executes children unconfined. Avoid.
- **Missing abstractions** such as `nameservice` (DNS and user lookups) or `ssl_certs`, causing strange denials.
- **Snap and container interplay.** Profiles for Snaps are generated by `snapd`. Do not edit them by hand.
- **Programs started from non-standard locations** (copies in `/opt`) are unconfined because the profile attaches to the path.
- **Confusing AppArmor with SELinux commands.** `setenforce` and `restorecon` do not apply.
- **Assuming a profile exists.** `aa-unconfined` lists exposed programs without protection.
- **`apparmor_parser` errors** after syntax changes cause the profile to not load. Check the message, and `aa-status` after reload.

## Further reading

- [AppArmor wiki](https://gitlab.com/apparmor/apparmor/-/wikis/home)
- [`apparmor(7)`](https://man7.org/linux/man-pages/man7/apparmor.7.html)
- [`apparmor.d(5)`](https://manpages.ubuntu.com/manpages/noble/man5/apparmor.d.5.html)
- [Ubuntu Server: AppArmor](https://documentation.ubuntu.com/server/how-to/security/apparmor/)
- [SUSE: AppArmor guide](https://documentation.suse.com)
