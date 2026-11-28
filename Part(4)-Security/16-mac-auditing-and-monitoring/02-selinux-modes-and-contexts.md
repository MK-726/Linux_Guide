# SELinux: Modes and Contexts

## Overview

**SELinux** (Security-Enhanced Linux) is the MAC system on RHEL-based distributions and Fedora. Every process and every file, port, and socket has a **security context (label)**. A **policy** states which labels may access which others. This page covers modes and how to read labels, which is the foundation for everything else in SELinux.

## Key concepts

### Modes

| Mode | Meaning |
|------|---------|
| **Enforcing** | Policy is enforced. Denied actions fail and are logged |
| **Permissive** | Policy is **not** enforced, but denials are logged (as "would have been denied") |
| **Disabled** | SELinux is off |

Two levels of configuration:

| Setting | Where | Effect |
|---------|-------|--------|
| **Runtime mode** | `setenforce 0` (permissive) or `setenforce 1` (enforcing) | Temporary. Lost at reboot |
| **Boot mode** | `/etc/selinux/config`: `SELINUX=` set to `enforcing`, `permissive`, or `disabled` | Used at the next boot |
| **Kernel command line** | `enforcing=0` or `selinux=0` | Overrides for one boot (recovery or debugging) |

`/etc/selinux/config`:

```
SELINUX=enforcing
SELINUXTYPE=targeted
```

### Policy type

| Type | Meaning |
|------|---------|
| **targeted** | **Default.** Confines selected network-facing and system services. Ordinary user logins run mostly unconfined |
| **mls** | Multi-Level Security (government/military use) |

### The security context

Every object has a label with four fields:

```
user : role : type : level
system_u:system_r:httpd_t:s0
```

| Field | Meaning | Importance in targeted policy |
|-------|---------|-------------------------------|
| **user** | SELinux user (not the Linux user) | Low |
| **role** | Role (RBAC) | Low (`system_r` for services, `object_r` for files) |
| **type** | **The key field.** Determines what the object is and what it may do | **High**. Policy rules are written in terms of types |
| **level** | MLS/MCS sensitivity and categories (`s0`, `s0:c123,c456`) | Used for MCS isolation (containers, VMs) |

Naming conventions: user `_u`, role `_r`, type `_t`.

| Type | Used for |
|------|----------|
| `httpd_t` | The web server **process** (a "domain") |
| `httpd_sys_content_t` | Web content files (read-only for the web server) |
| `httpd_sys_rw_content_t` | Web content the server may write |
| `httpd_log_t` | Web server logs |
| `sshd_t` | The SSH daemon process |
| `ssh_home_t` | `~/.ssh` files |
| `user_home_t` | General files in home directories |
| `var_log_t` | General logs |
| `tmp_t` | `/tmp` |
| `etc_t` | `/etc` files |
| `shadow_t` | `/etc/shadow` |
| `http_port_t` | TCP ports 80, 443, 8080, ... |
| `ssh_port_t` | TCP port 22 |
| `container_t`, `container_file_t` | Container process and container files |
| `unconfined_t` | Unconfined user shells in the targeted policy |

### How a decision works

A process in domain `httpd_t` asks to read a file labeled `user_home_t`. The policy contains no rule allowing `httpd_t` to read `user_home_t`, so the access is **denied** (and logged), even if the file permissions allow it. This is **type enforcement**.

## Commands

### Status and mode

```console
$ getenforce
Enforcing
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      33

# setenforce 0                      # permissive until reboot (troubleshooting)
# setenforce 1                      # enforcing again
$ cat /etc/selinux/config
```

### See labels

```console
$ ls -Z /var/www/html /etc/shadow ~/
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html
system_u:object_r:shadow_t:s0               /etc/shadow
$ ls -dZ /tmp /var/log
$ ps -eZ | grep -E 'sshd|nginx|httpd|chronyd'
system_u:system_r:sshd_t:s0-s0:c0.c1023 812 ? 00:00:00 sshd
system_u:system_r:httpd_t:s0     1020 ? 00:00:03 httpd
$ id -Z                                  # your own context
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
$ stat -c '%n %C' /etc/passwd            # context via stat
$ ss -tlnpZ | head                       # sockets with process context
```

### Inspect what the policy says

```console
# semanage port -l | grep -E '^http_port_t'
http_port_t     tcp   80, 81, 443, 488, 8008, 8009, 8443, 9000
# semanage fcontext -l | grep '/var/www'
/var/www(/.*)?       all files    system_u:object_r:httpd_sys_content_t:s0
# matchpathcon /var/www/html/index.html /home/alice/file.txt
/var/www/html/index.html   system_u:object_r:httpd_sys_content_t:s0
/home/alice/file.txt       system_u:object_r:user_home_t:s0
```

### Change labels

**Persistent method (use this):** define the rule, then apply it.

```console
# semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"     # add a path rule
# restorecon -Rv /srv/web                                          # apply labels according to rules
Relabeled /srv/web/index.html from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
```

**Temporary method (testing only):**

```console
# chcon -t httpd_sys_content_t /srv/web/index.html
# chcon -R --reference=/var/www/html /srv/web
```

`chcon` changes are **lost** at a full relabel or `restorecon`. Always add an `fcontext` rule for permanent changes.

### File label behavior to remember

| Operation | Resulting label |
|-----------|-----------------|
| **Create** a new file | Inherits from the directory (or the policy's default for that location) |
| **`cp`** | New file → gets the **destination's** default label |
| **`mv`** | The file **keeps its old label** (wrong in the new place). Run `restorecon` after moving |
| **`tar`/`rsync`** | Labels not preserved unless `--selinux` / `-X` flags are used |

### Switch modes safely

**Enforcing → permissive for a test:** `setenforce 0`, test, check logs, `setenforce 1`.

**Disabled → enforcing:** the filesystem has no labels, so a full relabel is needed.

```console
# vi /etc/selinux/config                 # SELINUX=permissive first (not enforcing!)
# touch /.autorelabel
# reboot                                 # relabels all files (can take a long time); then check the logs
# ausearch -m AVC -ts boot               # review denials
# vi /etc/selinux/config                 # SELINUX=enforcing
# reboot
```

Going to permissive first lets you find problems without breaking boot.

### Per-domain permissive mode

Instead of making the whole system permissive, relax one domain:

```console
# semanage permissive -a httpd_t         # httpd_t denials are logged but not enforced
# semanage permissive -l
# semanage permissive -d httpd_t         # remove
```

### SELinux users and logins (briefly)

```console
# semanage login -l                      # Linux user → SELinux user mapping
Login Name  SELinux User  MLS/MCS Range
__default__ unconfined_u  s0-s0:c0.c1023
root        unconfined_u  s0-s0:c0.c1023
# semanage user -l
# semanage login -a -s user_u alice      # confine a user (they can no longer use sudo/setuid by default, for restricted accounts)
```

### Install management tools

```console
# dnf install policycoreutils policycoreutils-python-utils setools-console setroubleshoot-server selinux-policy-doc
```

`man httpd_selinux`, `man ftpd_selinux`, `man samba_selinux`, etc. (from `selinux-policy-doc`) describe each service's types and booleans. `sepolicy manpage -a` generates them.

## Common pitfalls

- **`SELINUX=disabled` as a fix.** Re-enabling later requires a full relabel and may surface many problems. Use permissive for testing.
- **Using `chcon` and expecting permanence.** Use `semanage fcontext` + `restorecon`.
- **Moving files with `mv`** into a service directory. They keep the old label. Run `restorecon -Rv`.
- **Forgetting that `setenforce` is temporary.** The config file controls the next boot.
- **Relabeling the wrong path** with `restorecon -R /` unintentionally (takes long and can reset intentional custom labels).
- **Putting data in non-standard directories** (`/data/mysql`, `/opt/app/data`) without labels. The service gets denied.
- **Switching between `enforcing` and `disabled` without a relabel** can leave files unlabeled and produce boot or login failures.
- **Assuming SELinux is a "RHEL thing" only.** It exists on Debian and others but requires installing policy and setting up labels. Vendor support matters.
- **Reading only the type and ignoring the MCS level** (`c123,c456`) in containers and VMs. Different categories isolate them from each other.

## Further reading

- [Red Hat: using SELinux](https://docs.redhat.com)
- [`selinux(8)`](https://man7.org/linux/man-pages/man8/selinux.8.html)
- [`semanage(8)`](https://man7.org/linux/man-pages/man8/semanage.8.html)
- [`restorecon(8)`](https://man7.org/linux/man-pages/man8/restorecon.8.html)
- [SELinux Notebook](https://github.com/SELinuxProject/selinux-notebook)
