# sudoers Best Practices

## Overview

`sudo` lets selected users run commands as root (or another user) with logging. Used well, it implements **least privilege** and **accountability**. Used carelessly, it becomes a way to root. Basic usage is in [sudo and su](../../part-1-foundations/04-users-groups-and-permissions/07-sudo-and-su.md). This page covers **how to design and maintain** the rules.

## Key concepts

### Rule structure

```
WHO   WHERE=(RUNAS_USER:RUNAS_GROUP)   [TAGS:]  WHAT
```

| Part | Example |
|------|---------|
| WHO | `alice`, `%ops` (group), `+netgroup`, `User_Alias` |
| WHERE | `ALL`, `web01`, `Host_Alias` (hosts, useful when one file is shared) |
| RUNAS | `(root)`, `(postgres)`, `(ALL:ALL)` |
| TAGS | `NOPASSWD:`, `PASSWD:`, `NOEXEC:`, `SETENV:`, `LOG_INPUT:`, `LOG_OUTPUT:` |
| WHAT | `/usr/bin/systemctl restart nginx`, `Cmnd_Alias`, `ALL` |

**The last matching rule wins.** Put general rules first and exceptions last.

### Aliases

```
User_Alias   WEBADMINS = alice, bob
Host_Alias   WEBSERVERS = web01, web02
Cmnd_Alias   WEBSVC = /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx, /usr/bin/systemctl status nginx
WEBADMINS    WEBSERVERS=(root) WEBSVC
```

## Commands and recommended practices

### 1. Always use visudo and drop-in files

```console
# visudo                                  # edit /etc/sudoers with syntax checking
# visudo -f /etc/sudoers.d/10-webadmins   # a drop-in file
# visudo -c                               # check ALL sudoers files
# visudo -cf /etc/sudoers.d/10-webadmins  # check one file
```

- Put local rules in `/etc/sudoers.d/` (one file per team or purpose), not in the main file. Package updates may replace `/etc/sudoers`.
- Files must be `root:root` mode `0440`. Names must not contain `.` or end with `~` (they are ignored).
- The main file must contain `@includedir /etc/sudoers.d` (the `#includedir` form also works. The `#` there is **not** a comment).
- Keep drop-in files in version control or configuration management.

### 2. Grant specific commands, with full paths

```
# /etc/sudoers.d/20-ops
%ops  ALL=(root)  /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
%dba  ALL=(postgres)  /usr/bin/psql
```

A full-privilege rule (`ALL=(ALL) ALL`) should be reserved for a small administrator group (`%sudo`, `%wheel`).

### 3. Avoid shell escapes

Commands that can start a shell or run other programs give users **full root**, even if you intended something narrow.

| Dangerous to allow | Why |
|--------------------|-----|
| `vi`, `vim`, `nano`, `less`, `more`, `man`, `view` | Can run `!sh` |
| `find`, `awk`, `perl`, `python`, `ruby`, `node`, `lua`, `tar`, `zip`, `git`, `ftp`, `nmap`, `docker`, `systemctl edit`, `journalctl` (pager) | Can execute commands or write arbitrary files |
| `cp`, `mv`, `dd`, `tee`, `chmod`, `chown` with wildcards or no argument limits | Can overwrite `/etc/passwd`, `/etc/sudoers`, and other critical files |
| `bash`, `sh`, `su`, `sudo -i` | A root shell |
| `ALL` | Everything |
| Wildcards (`/usr/bin/systemctl restart *`) | `*` matches spaces and extra arguments |

Mitigations:

- Use **`sudoedit`** instead of letting users run an editor as root: `alice ALL=(root) sudoedit /etc/nginx/nginx.conf`.
- Add the **`NOEXEC`** tag so allowed programs cannot start other programs (works with dynamically linked programs):

  ```
  %support ALL=(root) NOEXEC: /usr/bin/less /var/log/syslog
  ```
- Allow **wrapper scripts** that you write and own (`root:root`, `755`, in a directory users cannot write) and that validate their own arguments.
- For `systemctl`, name exact commands rather than `systemctl *`.
- Check candidate commands against the **GTFOBins** list (`gtfobins.github.io`).

### 4. NOPASSWD sparingly

```
backup ALL=(root) NOPASSWD: /usr/local/bin/run-backup.sh
```

Use `NOPASSWD` only for narrow, non-interactive automation commands. Never for `ALL`.

### 5. Recommended Defaults

```
# /etc/sudoers.d/00-defaults
Defaults  use_pty
Defaults  env_reset
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults  timestamp_timeout=5
Defaults  passwd_tries=3
Defaults  !visiblepw
Defaults  logfile="/var/log/sudo.log"
Defaults  log_input, log_output
Defaults  iolog_dir="/var/log/sudo-io/%{user}"
Defaults  mail_badpass
Defaults  lecture=once
```

| Default | Purpose |
|---------|---------|
| `use_pty` | Run commands in a pseudo-terminal (stops some attacks on the terminal) |
| `env_reset` | Clean the environment (default). Do not weaken it |
| `secure_path` | Fixed `PATH` for commands |
| `timestamp_timeout` | Minutes `sudo` remembers the password (0 = always ask) |
| `logfile` | A separate sudo log in addition to the system log |
| `log_input`, `log_output` | **Record terminal sessions** for later replay (`sudoreplay`) |
| `env_keep` | Environment variables to keep (add sparingly) |
| `requiretty` | Old setting. Not recommended now (breaks automation). Prefer `use_pty` |

### 6. Audit what sudo did

```console
$ sudo -l                                         # your own rules
$ sudo -l -U alice                                # (root) rules for a user
# journalctl _COMM=sudo --since today
# grep -E 'COMMAND=' /var/log/auth.log            # Debian-based; RHEL-based: /var/log/secure
# sudoreplay -l                                   # list recorded sessions (with log_output)
# sudoreplay -d /var/log/sudo-io TSID             # replay one session
# sudo -V | head -3                               # sudo version
```

Send sudo logs to your central log system. Alert on use by unexpected users, use of dangerous commands, and failed attempts.

### 7. Centralize in larger environments

Store rules in **LDAP / FreeIPA / AD** and read them with SSSD, so every server has the same rules:

```
# /etc/nsswitch.conf
sudoers: files sss
```

```
# /etc/sssd/sssd.conf
[sssd]
services = nss, pam, sudo
```

Alternatively, distribute `/etc/sudoers.d/` with configuration management (Ansible, Puppet). See [SSSD and Centralized Identity Management](10-sssd-and-centralized-identity-management.md).

### 8. Group-based administration

Add people to an administrator group (`sudo` on Debian-based, `wheel` on RHEL-based) rather than writing a rule per user. Remove them when they leave, and review membership.

```console
# usermod -aG sudo alice
$ getent group sudo wheel
```

### 9. Prefer sudo to shared root passwords

- Lock the direct root password where possible (`passwd -l root`) and prevent root SSH login.
- Keep a **documented break-glass** method (console access, vaulted credentials).

### 10. Test safely

Before closing your session after changes:

```console
$ sudo -v                                    # in a second terminal, confirm sudo still works
# visudo -c                                  # syntax OK?
```

## Common pitfalls

- **Editing `/etc/sudoers` with a normal editor.** A syntax error disables sudo for everyone. Use `visudo`.
- **Drop-in file mode wrong or name with a dot.** The file is ignored or rejected.
- **`ALL` or wildcards in the command list** defeating least privilege.
- **Allowing editors, pagers, or interpreters.** They provide root shells.
- **Relative paths** in rules. Always use absolute paths.
- **Negation (`!`) as a security measure**, for example `ALL, !/bin/su`. It is easily bypassed by copying the binary. Allow only what is needed instead.
- **Rule order mistakes** (the last match wins).
- **`env_keep` of `LD_PRELOAD`, `PATH`, `PYTHONPATH`** and similar variables allows code injection.
- **`NOPASSWD: ALL`** for convenience.
- **Never reviewing rules.** Remove obsolete entries. Compare `sudo -l -U` with job roles.
- **Not logging sessions.** Without `log_output` you only know the command, not what a shell session did.
- **Using `sudo -s` or `sudo su -`** habitually: you lose per-command logging in the log file (session recording helps).
- **Hostname-based rules** that break when host names change or when the same file is used on cloned systems.
- **Assuming sudo replaces other controls.** Pair it with MFA on login, SSH restrictions, and MAC.

## Further reading

- [`sudoers(5)`](https://www.sudo.ws/docs/man/sudoers.man/)
- [`visudo(8)`](https://man7.org/linux/man-pages/man8/visudo.8.html)
- [`sudoreplay(8)`](https://www.sudo.ws/docs/man/sudoreplay.man/)
- [GTFOBins](https://gtfobins.github.io)
- [Sudo project documentation](https://www.sudo.ws/docs/)
