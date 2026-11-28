# Discretionary and Mandatory Access Control

## Overview

Standard Linux permissions are **DAC** (Discretionary Access Control). They are simple and flexible, but they have a weakness: a process runs with **all the rights of its user**. If an attacker compromises a web server running as `www-data`, they can read everything `www-data` can read. **MAC** (Mandatory Access Control) adds a **system-wide policy** that confines each program to what it needs, regardless of the user.

## Key concepts

### DAC vs. MAC

| | **DAC** | **MAC** |
|---|---------|---------|
| Who decides | The **owner** of the object (and root) | The **security policy** set by the administrator |
| Based on | User, group, mode bits, ACLs | Labels or profiles of **processes** and **objects** |
| Can a user change access? | Yes (`chmod`, `chown`) | **No**, not without policy permission |
| Does root bypass it? | Yes, mostly | **No**. Confined root processes are still limited |
| Granularity | Per file, per user | Per program, per resource, per operation |
| Example | `-rw-r----- root www-data` | "`httpd_t` may read `httpd_sys_content_t`, and may not read `/etc/shadow`" |

Both are checked. **An access must be allowed by DAC and by MAC.** MAC can only **deny more**, never grant what DAC denies.

### An example

An attacker exploits a bug in a web application and gets a shell as the web server user.

| With DAC only | With MAC (SELinux/AppArmor) |
|---------------|-----------------------------|
| Reads any file readable by that user (SSH keys, configuration, other apps' data), connects anywhere, can run tools | The web server process is confined: it can read web content, write its logs, and talk to its own database port. Reading `/home`, starting shells with privileges, or opening other ports is **denied** and **logged** |

### Linux Security Modules (LSM)

MAC in Linux is implemented through the **LSM framework**, which lets the kernel ask a module "may this process do this?" at many points.

| Module | Type | Typical use |
|--------|------|-------------|
| **SELinux** | Label-based (type enforcement) | RHEL, Fedora, Rocky, AlmaLinux, Android. Very fine-grained |
| **AppArmor** | **Path-based** profiles | Ubuntu, Debian (available), SUSE (SLES 15, openSUSE) |
| **SMACK** | Label-based, simple | Embedded, Tizen |
| **TOMOYO** | Path-based, learning mode | Rare |
| **Yama** | Restricts `ptrace` | Enabled by default on many systems |
| **Landlock** | Unprivileged self-sandboxing | Applications restrict themselves |
| **BPF LSM** | Policy in eBPF | Newer, advanced |
| **Lockdown, IPE, capabilities** | Various | Part of the stack |

Several LSMs can be active at once. Only one "major" MAC (SELinux or AppArmor) is normally used.

```console
$ cat /sys/kernel/security/lsm
lockdown,capability,landlock,yama,apparmor,ima,evm
$ cat /sys/kernel/security/lsm            # on RHEL-based:
lockdown,capability,yama,selinux,bpf,landlock,ima,evm
```

### SELinux vs. AppArmor

| | **SELinux** | **AppArmor** |
|---|-------------|--------------|
| Model | **Labels** (type enforcement) on every process and object | **Profiles** attached to program **paths** |
| Identification of files | Label stored in an extended attribute, follows the file | By path name |
| Policy size | Large and detailed | Smaller and easier to read |
| Learning curve | Steeper | Gentler |
| Flexibility | Very strong, covers networking, users, MLS/MCS | Good for application confinement |
| Default on | RHEL-based, Fedora | Ubuntu, SUSE (SLES 15, openSUSE) |
| Containers | MCS labels isolate containers (`container_t`) | Profiles per container |
| Main tools | `getenforce`, `semanage`, `restorecon`, `ausearch`, `sealert` | `aa-status`, `aa-genprof`, `aa-logprof`, `apparmor_parser` |

Both are valid. **Use the one your distribution supports and maintains.** Running SELinux on Ubuntu or AppArmor on RHEL means little vendor policy and support.

### How MAC decisions look

- **SELinux** is **default deny**: every access needs an explicit **allow rule** (`allow httpd_t httpd_sys_content_t:file { read open getattr };`).
- **AppArmor** profiles **list** allowed paths and capabilities. Everything else is denied for confined programs. Programs **without** a profile are **unconfined**.

### Modes

| Mode | Behavior |
|------|----------|
| **Enforcing** | Violations are **blocked and logged** |
| **Permissive / complain** | Violations are **allowed but logged** (for developing policy and troubleshooting) |
| **Disabled** | Not active |

## Commands

```console
$ getenforce 2>/dev/null                        # SELinux mode (RHEL-based)
Enforcing
$ sudo aa-status 2>/dev/null | head -5          # AppArmor (Ubuntu, SUSE)
apparmor module is loaded.
50 profiles are loaded.
48 profiles are in enforce mode.
$ ps -eZ | head -3                               # SELinux label of processes
$ ps auxZ | grep -E 'nginx|httpd'
$ cat /proc/$$/attr/current                      # label/profile of the current shell
```

### Verify that MAC is working for a service

```console
$ ps -eZ | grep nginx                            # SELinux: system_u:system_r:httpd_t:s0 ... nginx
$ sudo aa-status | grep -i nginx                 # AppArmor: listed in enforce mode?
```

If the service runs as `unconfined_service_t` (SELinux) or appears under "processes that are unconfined" (AppArmor), it is **not protected** by MAC.

## Common pitfalls

- **Thinking permissions alone are enough.** Compromised services use their user's full rights without MAC.
- **Disabling MAC at the first denial.** Fix it properly (Chapters 2 to 5 of this section).
- **Mixing assumptions between SELinux and AppArmor.** Labels vs. paths behave differently (moving a file keeps its SELinux label but changes the AppArmor path).
- **Using MAC without logging and review.** Denials show attacks and misconfigurations. Monitor them.
- **Thinking MAC makes bugs harmless.** It limits impact. Patching still matters.
- **Running services in unconfined domains** (custom software installed in `/opt`) and assuming they are protected.
- **Assuming containers are isolated** by MAC alone. It is one layer among namespaces, cgroups, capabilities, and seccomp.

## Further reading

- [Linux Security Modules (kernel docs)](https://www.kernel.org/doc/html/latest/admin-guide/LSM/index.html)
- [SELinux project](https://github.com/SELinuxProject)
- [AppArmor wiki](https://gitlab.com/apparmor/apparmor/-/wikis/home)
- [NSA: original SELinux paper / background](https://www.nsa.gov/Research/Technical-Papers/)
