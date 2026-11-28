# PAM: Pluggable Authentication Modules

## Overview

**PAM** is the framework most Linux programs use for authentication and account checks. Programs such as `login`, `sshd`, `sudo`, `su`, and `passwd` do not handle passwords themselves. They call PAM, which runs a **stack of modules** configured by the administrator. By editing the PAM configuration you can add password rules, account lockout, MFA, home directory creation, central identity (SSSD), and more, **without changing the programs**.

> [!WARNING]
> A mistake in PAM can lock **everyone** out, including root. Keep a root shell open while testing, use the console, and keep a backup of `/etc/pam.d/`.

## Key concepts

### The four management groups

Every PAM rule belongs to one of four types:

| Type | Question answered | Example |
|------|-------------------|---------|
| `auth` | **Who are you?** Verifies the user (password, OTP, key) | Check the password |
| `account` | **Is this account allowed now?** Expiry, time and host restrictions, locked | Account expired? Allowed from this host? |
| `password` | **Changing the credential** Password strength and update | Enforce complexity on `passwd` |
| `session` | **Set up and tear down the session** Limits, mount, logging, home directory | Apply `ulimit`, create the home directory, register with systemd |

### Rule format

```
type    control    module-path    [module-arguments]
```

Example from `/etc/pam.d/sshd`:

```
auth       required     pam_unix.so nullok
account    required     pam_nologin.so
session    optional     pam_motd.so
```

### Control flags

The **control** decides what happens when a module succeeds or fails:

| Control | Meaning |
|---------|---------|
| `required` | Must succeed. On failure the stack **continues** (so an attacker cannot tell which module failed) and the final result is failure |
| `requisite` | Must succeed. On failure the stack **stops immediately** and returns failure |
| `sufficient` | If it succeeds (and no earlier `required` failed), the stack **stops with success**. On failure it is ignored |
| `optional` | Result matters only if it is the only module in the stack |
| `include` | Include all lines of the same type from another file |
| `substack` | Like `include`, but its result is evaluated as one unit |
| `[value=action ...]` | Advanced syntax, for example `[success=1 default=ignore]` (skip the next rule on success) and `[default=die]` |

The order of lines **matters**. PAM reads each type's rules from top to bottom.

### Configuration files

| Path | Purpose |
|------|---------|
| `/etc/pam.d/SERVICE` | One file per service (`sshd`, `sudo`, `login`, `su`, `passwd`, `cron`, `gdm`) |
| `/etc/pam.conf` | Old single-file format (rarely used) |
| `/etc/security/` | Configuration for modules (`limits.conf`, `access.conf`, `time.conf`, `pwquality.conf`, `faillock.conf`) |
| `/lib/security/` or `/usr/lib/x86_64-linux-gnu/security/` or `/usr/lib64/security/` | Module files (`pam_*.so`) |

### Shared stack files by distribution

Most services include **shared** stacks, so one change applies to all of them.

| Family | Shared files | Managed with |
|--------|--------------|--------------|
| Debian-based | `common-auth`, `common-account`, `common-password`, `common-session`, `common-session-noninteractive` | **`pam-auth-update`** (profiles in `/usr/share/pam-configs/`). Do not edit by hand if you can use a profile |
| RHEL-based | `system-auth`, `password-auth`, `smartcard-auth`, `fingerprint-auth` | **`authselect`** (RHEL 8 and newer; replaced `authconfig`). Files carry a "managed by authselect" header. Manual edits are overwritten or rejected |
| SUSE-based | `common-auth`, `common-account`, `common-password`, `common-session` | `pam-config` |

### Common modules

| Module | Purpose |
|--------|---------|
| `pam_unix` | Traditional passwords (`/etc/shadow`). Options: `nullok` (allow empty passwords, **avoid**), `sha512`/`yescrypt`, `remember=N` |
| `pam_sss` | Authenticate through SSSD (LDAP, AD, FreeIPA) |
| `pam_faillock` | Lock the account after repeated failures (replaces `pam_tally2`) |
| `pam_pwquality` | Password strength rules (replaces `pam_cracklib`) |
| `pam_limits` | Applies `/etc/security/limits.conf` resource limits |
| `pam_access` | Allow or deny by user, group, origin (`/etc/security/access.conf`) |
| `pam_time` | Time-of-day restrictions |
| `pam_wheel` | Only members of a group may use `su` |
| `pam_nologin` | Block non-root logins when `/etc/nologin` exists |
| `pam_listfile` | Allow or deny from a list |
| `pam_succeed_if` | Conditions on user, UID, group, shell |
| `pam_env` | Set environment variables |
| `pam_systemd` | Register the login session with `systemd-logind` |
| `pam_mkhomedir`, `pam_oddjob_mkhomedir` | Create a home directory at first login |
| `pam_selinux` | Set the SELinux context of the session |
| `pam_google_authenticator` | TOTP second factor |
| `pam_u2f` | FIDO/U2F security keys |
| `pam_deny`, `pam_permit` | Always deny / always allow (used as stack end markers) |
| `pam_motd`, `pam_lastlog` | Messages and last login information |

## Commands

### Look at the configuration

```console
$ cat /etc/pam.d/sshd
$ cat /etc/pam.d/common-auth                 # Debian-based
$ cat /etc/pam.d/system-auth                 # RHEL-based
$ authselect current                         # RHEL-based: active profile and features
Profile ID: sssd
Enabled features:
- with-mkhomedir
$ authselect list
$ pam-auth-update --package                  # Debian-based (interactive menu without options)
$ ls /usr/share/pam-configs/                 # Debian-based profiles
$ man pam.d
$ man pam_faillock
```

### Change PAM safely: tools

```console
# authselect select sssd with-mkhomedir with-faillock --force      # RHEL-based
# authselect enable-feature with-faillock
# authselect apply-changes
# pam-auth-update --enable mkhomedir                               # Debian-based
# pam-auth-update --remove unix
```

Use these tools instead of editing the generated files. To customize on RHEL-based systems, create a custom authselect profile (`authselect create-profile`).

### Account lockout with pam_faillock

On current RHEL-based systems, set options in `/etc/security/faillock.conf` and enable the feature:

```
# /etc/security/faillock.conf
deny = 5
fail_interval = 900
unlock_time = 900
even_deny_root
root_unlock_time = 60
audit
silent
```

```console
# authselect enable-feature with-faillock                          # RHEL-based
$ faillock --user alice                                            # show failures
# faillock --user alice --reset                                    # unlock
```

| Setting | Meaning |
|---------|---------|
| `deny` | Failures before the account is locked |
| `fail_interval` | Window (seconds) in which failures are counted |
| `unlock_time` | Seconds the account stays locked (`0` = until an administrator resets it) |
| `even_deny_root` | Apply the lockout to root too (**be careful: it can enable a denial of service**) |

Lockout protects against password guessing but lets an attacker **lock out legitimate users**. Use a reasonable `unlock_time`, rate-limit at the network level, and prefer key-only SSH for sensitive hosts.

### Test a PAM stack without logging in

```console
$ sudo apt install pamtester          # or: dnf install pamtester (EPEL)
$ pamtester sshd alice authenticate   # prompts for the password and reports the result
$ pamtester sshd alice acct_mgmt
```

### Typical edits explained

Restrict `su` to the `wheel` group (`/etc/pam.d/su`):

```
auth    required    pam_wheel.so use_uid group=wheel
```

Limit SSH logins by origin and group with `pam_access` (`/etc/pam.d/sshd` and `/etc/security/access.conf`):

```
account required pam_access.so
```

```
# /etc/security/access.conf  (first match wins)
+ : root : 192.0.2.0/24 LOCAL
+ : @sshusers : ALL
- : ALL : ALL
```

Password quality for `passwd` (see [Password Policies and Aging](04-password-policies-and-aging.md)):

```
password requisite pam_pwquality.so retry=3
password sufficient pam_unix.so use_authtok yescrypt shadow
```

Create the home directory at the first login (central users):

```
session optional pam_mkhomedir.so skel=/etc/skel umask=0077
```

### Diagnose PAM problems

```console
$ journalctl -t sshd -t sudo -t su -t login --since "10 min ago"
$ grep -i pam /var/log/auth.log | tail                    # Debian-based
$ grep -i pam /var/log/secure | tail                      # RHEL-based
$ strace -f -e trace=openat -o /tmp/pam.trace su - alice  # which files and modules are used
```

Add `debug` to a module line (for example `auth required pam_unix.so debug`) to log more detail. Remove it afterwards.

## Common pitfalls

- **Locking yourself out.** Test with a second session, pamtester, and console access. Back up `/etc/pam.d` before editing (`cp -a /etc/pam.d /root/pam.d.bak`).
- **Wrong order of lines.** For example, a `sufficient` rule placed before a `required` check lets users skip the check.
- **Editing generated files** managed by `authselect` or `pam-auth-update`. Changes are overwritten. Use the tools.
- **`nullok`** allows empty passwords. Remove it in production.
- **Using `required` where `requisite` is wanted** (or the reverse) gives surprising behavior and leaks information.
- **Forgetting that `sudo`, `su`, `cron`, `sshd`, and the console have separate PAM files.** A rule added to `common-auth` affects them all, but a rule in `/etc/pam.d/sshd` affects only SSH.
- **Missing `@include common-auth` in `sshd` while enabling TOTP**, or the reverse. See [Multi-Factor Authentication Concepts](07-multi-factor-authentication-concepts.md).
- **`pam_tally2`** is removed from recent distributions. Use `pam_faillock`.
- **Lockout on root with `even_deny_root`** during an attack makes remote repair harder.
- **Not restarting long-running services.** Most services read PAM files at each authentication, but daemons that cache (some desktop managers) may need a restart.
- **SELinux** may block PAM modules that read files in non-standard places. Check `ausearch -m avc`.

## Further reading

- [`pam(8)`](https://man7.org/linux/man-pages/man8/pam.8.html)
- [`pam.d(5)`](https://man7.org/linux/man-pages/man5/pam.d.5.html)
- [Linux-PAM System Administrators' Guide](https://www.chiark.greenend.org.uk/doc/libpam-doc/html/Linux-PAM_SAG.html)
- [`pam_faillock(8)`](https://man7.org/linux/man-pages/man8/pam_faillock.8.html)
- [`authselect(8)`](https://man7.org/linux/man-pages/man8/authselect.8.html)
