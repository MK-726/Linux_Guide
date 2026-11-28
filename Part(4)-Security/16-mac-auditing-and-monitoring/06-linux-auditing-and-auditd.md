# Linux Auditing and auditd

## Overview

The **Linux Audit system** records security-relevant events from the **kernel**: file access, system calls, logins, privilege use, and policy violations (SELinux AVCs). The user-space daemon **`auditd`** writes them to `/var/log/audit/audit.log`. Auditing answers *"who did what, when, and from where?"* and is often required for compliance (PCI DSS, STIG, HIPAA).

## Key concepts

### Pieces

| Component | Role |
|-----------|------|
| **Kernel audit subsystem** | Generates events according to rules |
| **`auditd`** | Receives events and writes the log |
| **`auditctl`** | Loads and inspects rules in the running kernel |
| **`/etc/audit/rules.d/*.rules`** | Persistent rules. `augenrules` merges them into `/etc/audit/audit.rules` |
| **`/etc/audit/auditd.conf`** | Daemon settings (log location, size, rotation, actions) |
| **`ausearch`, `aureport`, `aulast`** | Search and summarize logs |
| **audisp plugins** (`/etc/audit/plugins.d/`) | Forward events (remote, syslog) |

### The audit user ID (auid)

The **audit login UID (`auid`)** is set at login and **does not change** when the user runs `sudo` or `su`. So an action by root after `sudo` is still attributed to the original person (`auid=1000`). Systems service processes have `auid=unset` (`4294967295`).

### Rule types

| Type | Option | Example |
|------|--------|---------|
| **File watch** | `-w PATH -p PERMS -k KEY` | Watch `/etc/passwd` for writes and attribute changes |
| **System call rule** | `-a always,exit -F arch=b64 -S SYSCALL -k KEY` | Record every `execve` or `unlink` |
| **Control rule** | `-D`, `-b`, `-f`, `-e` | Delete rules, backlog size, failure mode, enable/immutable |

Watch permissions: `r` read, `w` write, `x` execute, `a` attribute change.

A **key** (`-k name`) labels events so you can search them easily.

## Commands

### Install and enable

```console
# apt install auditd audispd-plugins                 # Debian-based
# dnf install audit                                  # RHEL-based (installed by default)
# systemctl enable --now auditd
# auditctl -s                                        # status
enabled 1
failure 1
pid 812
backlog_limit 8192
lost 0
backlog 0
# auditctl -l                                        # loaded rules
```

> [!NOTE]
> On RHEL-based systems, `auditd` cannot be restarted with `systemctl restart auditd`. Use `service auditd restart` (or `auditctl -R` / `augenrules --load` to reload rules).

### Temporary rules with auditctl

```console
# auditctl -w /etc/passwd -p wa -k identity                      # watch writes and attribute changes
# auditctl -w /etc/sudoers.d/ -p wa -k sudoers
# auditctl -a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=unset -k user_exec    # commands run by real users
# auditctl -a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat -F auid>=1000 -F auid!=unset -k delete
# auditctl -d always,exit ...                                    # delete a specific rule
# auditctl -D                                                    # delete all rules
```

These are lost at reboot unless saved in a rules file.

### Persistent rules

Create `/etc/audit/rules.d/50-security.rules` (files are loaded in order):

```
## Remove any existing rules and set the buffer
-D
-b 8192
-f 1

## Identity and authentication
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers
-w /etc/pam.d/ -p wa -k pam
-w /etc/ssh/sshd_config -p wa -k sshd
-w /etc/ssh/sshd_config.d/ -p wa -k sshd

## Login records
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins

## Time changes
-a always,exit -F arch=b64 -S adjtimex,settimeofday,clock_settime -k time-change

## Kernel modules
-a always,exit -F arch=b64 -S init_module,finit_module,delete_module -k modules
-w /sbin/insmod -p x -k modules
-w /sbin/modprobe -p x -k modules

## Mounts
-a always,exit -F arch=b64 -S mount,umount2 -F auid>=1000 -F auid!=unset -k mounts

## Privilege use and commands from real users
-a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=unset -k user_exec
-a always,exit -F arch=b64 -C euid!=uid -F euid=0 -F auid>=1000 -F auid!=unset -S execve -k priv_exec

## File deletion and permission changes by users
-a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat -F auid>=1000 -F auid!=unset -k delete
-a always,exit -F arch=b64 -S chmod,fchmod,fchmodat,chown,fchown,lchown,fchownat -F auid>=1000 -F auid!=unset -k perm_mod

## Persistence locations
-w /etc/cron.d/ -p wa -k cron
-w /etc/crontab -p wa -k cron
-w /var/spool/cron/ -p wa -k cron
-w /etc/systemd/system/ -p wa -k systemd
-w /root/.ssh/ -p wa -k ssh_keys

## Make the configuration immutable (requires reboot to change). Keep this LAST.
#-e 2
```

Apply:

```console
# augenrules --check                  # do the files differ from the loaded rules?
# augenrules --load                   # merge and load
# auditctl -l | head
```

> [!WARNING]
> `-e 2` locks the audit configuration until reboot. It protects against tampering but also stops you from fixing mistakes. Test the rule set first, then enable it. Add rules for **both** `arch=b64` and, on systems that run 32-bit programs, `arch=b32`.

CIS and STIG provide complete rule sets. Distributions ship samples in `/usr/share/audit/sample-rules/` or `/usr/share/doc/audit*/rules/`.

### Search events

```console
# ausearch -k identity -i                              # by key, human-readable
# ausearch -k user_exec -ts today -i | tail
# ausearch -m USER_LOGIN -sv no -ts recent -i          # failed logins
# ausearch -ua alice -ts yesterday -te today -i        # by user
# ausearch -x /usr/bin/sudo -i                         # by executable
# ausearch -f /etc/shadow -i                           # by file
# ausearch -m ADD_USER,DEL_USER,ADD_GROUP -i           # account changes
# ausearch -m AVC -ts recent -i                        # SELinux denials
# ausearch --start 03/03/2025 09:00:00 --end 03/03/2025 10:00:00 -i
```

Reading a record (`-i` interprets IDs):

```
type=SYSCALL msg=audit(03/03/2025 10:15:22.123:789) : arch=x86_64 syscall=openat success=yes exit=3 a0=AT_FDCWD a1=0x7ffd... items=1 ppid=2101 pid=2150 auid=alice uid=root gid=root euid=root comm=vim exe=/usr/bin/vim key=identity
type=PATH msg=audit(03/03/2025 10:15:22.123:789) : item=0 name=/etc/passwd inode=131077 mode=file,644 ouid=root ogid=root
```

Interpretation: **alice** (`auid`) edited `/etc/passwd` with `vim` as **root** (through `sudo`). Both records share the same event ID (`789`).

### Reports

```console
# aureport --summary                                    # overview
# aureport -au --failed --summary                       # failed authentications
# aureport -l --failed -i                               # failed logins
# aureport -x --summary                                 # executables
# aureport -f -i --summary                              # files
# aureport -m -i                                        # account modifications
# aureport --key --summary                              # events per key
# aureport -t                                           # time range of the logs
# aulast | head                                         # logins from the audit log
```

### auditd.conf (log handling)

```
log_file = /var/log/audit/audit.log
log_format = ENRICHED
max_log_file = 50
max_log_file_action = ROTATE
num_logs = 10
space_left = 75
space_left_action = EMAIL
admin_space_left = 50
admin_space_left_action = HALT        # or SUSPEND: what to do when disk is nearly full (decide with your compliance needs)
disk_full_action = SUSPEND
flush = INCREMENTAL_ASYNC
```

Put `/var/log/audit` on its **own partition** so a full disk does not affect the system. `HALT` stops the system when audit space is exhausted. That ensures no unlogged activity but affects availability. Choose deliberately.

### Forward logs off the host

An attacker who gains root can edit local logs. Send them elsewhere:

- `audisp-remote` plugin with an `auditd` log server (`/etc/audit/plugins.d/au-remote.conf`).
- `rsyslog` imfile or the `syslog` audisp plugin to forward to a central server.
- Agents such as Filebeat, Wazuh agent, Splunk forwarder.

### Performance

Auditing every `execve` and file operation produces a lot of data. Tune:

- Audit by **key paths and risky syscalls**, not everything.
- Exclude noisy known-good activity: `-a never,exit -F arch=b64 -S execve -F exe=/usr/bin/backup-agent`. Place `never` rules **before** the rules they exclude from.
- Increase the backlog (`-b 8192` or more) and watch `lost` in `auditctl -s`.
- Use `ausyscall --dump` to find syscall names.

## Common pitfalls

- **Rules lost at reboot** because they were added with `auditctl` only. Use `/etc/audit/rules.d/`.
- **Only `arch=b64` rules** while 32-bit processes exist (or the reverse).
- **`auid!=unset` is not accepted by older audit versions.** Use `auid!=4294967295` there.
- **No central copy of logs.** Local audit logs can be modified by an intruder.
- **Disk full** from too much logging. Rotate, size correctly, and monitor.
- **Rule order.** Audit rules are evaluated in order and the **first match** wins for `exclude`/`never` rules. Put exclusions early.
- **`-e 2` enabled too early.** You cannot change rules without a reboot.
- **Auditing everything** and then ignoring the data. Define what you want to detect first.
- **`auid` confusion.** Daemon activity has `auid=unset`. Filter user activity with `auid>=1000`.
- **Missing the `-k` key** which makes searches hard.
- **Assuming auditd alone detects attacks.** It records. Detection requires someone or something to analyze the logs (SIEM, alerts).
- **Auditing inside containers.** The kernel audit system is global. Container activity appears with host IDs. Correlate with container metadata.

## Further reading

- [`auditd(8)`](https://man7.org/linux/man-pages/man8/auditd.8.html)
- [`audit.rules(7)`](https://man7.org/linux/man-pages/man7/audit.rules.7.html)
- [`ausearch(8)`](https://man7.org/linux/man-pages/man8/ausearch.8.html)
- [Linux Audit documentation project](https://github.com/linux-audit/audit-documentation)
- [Neo23x0 auditd rules](https://github.com/Neo23x0/auditd)
- [Red Hat: auditing the system](https://docs.redhat.com)
