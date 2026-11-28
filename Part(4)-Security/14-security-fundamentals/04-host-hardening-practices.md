# Host Hardening Practices

## Overview

**Hardening** means reducing a system's attack surface and tightening its configuration so that fewer things can go wrong, and the damage is limited when something does. This page gives a practical baseline, commands to **audit** it, and the tools that automate checks. Always test changes first, because aggressive settings can break applications.

## Key concepts

### Standards and benchmarks

| Standard | Description |
|----------|-------------|
| **CIS Benchmarks** | Detailed, consensus-based configuration guides per distribution (Level 1 = practical baseline, Level 2 = stricter) |
| **DISA STIG** | US Department of Defense configuration standards |
| **Vendor hardening guides** | Red Hat, Canonical, SUSE security guides |
| **NIST SP 800-53 / 800-171, PCI DSS, ISO 27001, HIPAA** | Frameworks that require hardening and evidence |
| **OpenSCAP and SCAP Security Guide** | Machine-readable profiles that **scan and remediate** (CIS, STIG, PCI-DSS, and more) |

Use a benchmark as the **baseline**, then adapt it to your applications and document exceptions.

### Hardening areas

| Area | Goal |
|------|------|
| Installation | Minimal packages, secure partitioning, disk encryption |
| Services | Only what is needed, correctly bound and sandboxed |
| Network | Default-deny firewall, hardened kernel network settings |
| Accounts | No shared or empty-password accounts, strong policy, MFA |
| Privileges | Least privilege, careful `sudo` |
| Files | Safe permissions, mount options |
| Kernel | Safe sysctl values, unused modules blocked |
| Updates | Patched promptly |
| Logging and auditing | Complete, protected, and monitored |
| MAC | SELinux or AppArmor enforcing |
| Boot and physical | Secure Boot, boot loader password, console security |

## Commands: a baseline, step by step

### 1. Minimal software and services

```console
# apt purge telnet rsh-client talk                   # Debian-based: remove legacy tools
# dnf remove telnet rsh talk                         # RHEL-based
$ systemctl list-unit-files --state=enabled --type=service
# systemctl disable --now cups avahi-daemon bluetooth    # example: only if not needed
# systemctl mask rpcbind.socket rpcbind.service      # if you do not use NFSv3 or NIS
$ ss -tulpn                                          # every listener must be justified
```

### 2. Accounts and authentication

```console
# awk -F: '$2=="" {print $1}' /etc/shadow            # no empty passwords
# awk -F: '$3==0 {print $1}' /etc/passwd             # only root has UID 0
# passwd -l unused_account                           # lock unused accounts
# usermod -s /usr/sbin/nologin svc_account           # no shell for service accounts
# chage -l alice                                     # password aging
```

Apply password policy, lockout, MFA, and `sudo` rules as covered in Chapter 15.

### 3. SSH

Key-only login, no direct root login, restricted users, limited forwarding (see [SSH Client and Server Configuration](../../part-3-networking/13-remote-access-firewalls-services/02-ssh-client-and-server-configuration.md)).

```console
# sshd -T | grep -E 'permitrootlogin|passwordauthentication|allowusers|allowgroups|x11forwarding|maxauthtries'
```

### 4. Firewall: default deny

See [Firewall Concepts](../../part-3-networking/13-remote-access-firewalls-services/05-firewall-concepts.md) and [nftables](../../part-3-networking/13-remote-access-firewalls-services/06-nftables.md).

### 5. Filesystem mount options

Use restrictive options on filesystems that do not need to run programs or create device files.

```
# /etc/fstab
tmpfs     /tmp        tmpfs  defaults,nodev,nosuid,noexec,size=2G   0 0
tmpfs     /dev/shm    tmpfs  defaults,nodev,nosuid,noexec           0 0
/dev/mapper/vg0-var     /var       xfs  defaults,nodev,nosuid        0 0
/dev/mapper/vg0-vartmp  /var/tmp   xfs  defaults,nodev,nosuid,noexec 0 0
/dev/mapper/vg0-home    /home      xfs  defaults,nodev,nosuid        0 0
```

| Option | Effect |
|--------|--------|
| `nodev` | Ignore device files |
| `nosuid` | Ignore SUID/SGID bits |
| `noexec` | Do not execute programs from this filesystem |

> [!NOTE]
> `noexec` on `/tmp` can break installers, some package scripts, and applications that unpack code there. Test first. It is a speed bump against attackers, not a complete barrier.

Separate partitions or volumes for `/var`, `/var/log`, `/var/log/audit`, `/var/tmp`, `/home`, and `/tmp` also prevent one filling the root filesystem.

```console
$ findmnt -no TARGET,OPTIONS /tmp /var/tmp /dev/shm /home
```

### 6. Kernel parameters

`/etc/sysctl.d/99-hardening.conf` (review each line before use):

```
# Network
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_ra = 0

# Kernel and memory
kernel.randomize_va_space = 2
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2
kernel.kexec_load_disabled = 1
kernel.sysrq = 0
fs.suid_dumpable = 0

# Filesystem
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
fs.protected_regular = 2
```

```console
# sysctl --system
```

> [!WARNING]
> Some settings affect function: routers need `ip_forward` and redirects behave differently, `accept_ra = 0` disables IPv6 autoconfiguration on that host, `ptrace_scope` can limit debuggers, `kernel.sysrq = 0` disables the emergency key. Adjust for your role.

### 7. Disable unused kernel modules and protocols

```
# /etc/modprobe.d/hardening.conf
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
install cramfs /bin/false
install usb-storage /bin/false      # only on servers that never need USB storage
blacklist firewire-core
```

### 8. Core dumps

Core dumps can contain secrets.

```console
# echo '* hard core 0' > /etc/security/limits.d/10-no-core.conf
# mkdir -p /etc/systemd/coredump.conf.d && printf '[Coredump]\nStorage=none\nProcessSizeMax=0\n' > /etc/systemd/coredump.conf.d/disable.conf
```

(`fs.suid_dumpable = 0` from the sysctl file also helps.) Keep dumps only where you actively debug.

### 9. Warning banner, time, and logging

```console
# echo "Authorized use only. Activity is monitored and logged." > /etc/issue.net
```

Set `Banner /etc/issue.net` in `sshd_config`. Keep clocks synchronized ([NTP and chrony](../../part-2-system-administration/10-logging-time-and-scheduling/04-time-zones-ntp-and-chrony.md)). Send logs to a central server and enable `auditd` (Chapter 16).

### 10. Mandatory access control

Keep **SELinux enforcing** (RHEL-based) or **AppArmor enforcing** (Ubuntu, SUSE). See Chapter 16.

```console
$ getenforce            # Enforcing
$ aa-status | head -5
```

### 11. Boot and physical

UEFI Secure Boot, firmware password, GRUB password, disk encryption (see [Secure Boot Overview](07-secure-boot-overview.md)).

### 12. Compliance and audit tools

**Lynis** (quick, general audit):

```console
# apt install lynis        # or: dnf install lynis (EPEL)
# lynis audit system
...
  Hardening index : 68 [#############       ]
  Tests performed : 255
Suggestions (22):
  * Configure password hashing rounds in /etc/login.defs [AUTH-9230]
  * Install a PAM module for password strength testing like pam_cracklib or pam_passwdqc [AUTH-9262]
```

**OpenSCAP** (benchmark-based scanning and remediation):

```console
# dnf install openscap-scanner scap-security-guide             # RHEL-based
# apt install openscap-scanner ssg-base ssg-debderived         # Debian-based (package names vary by release)
# oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml  # list profiles
# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
      --results /tmp/results.xml --report /tmp/report.html \
      /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

Open `report.html` to see passed and failed rules, and the reason for each. SCAP content can also **generate remediation scripts or Ansible playbooks** (`oscap xccdf generate fix`). Review them before running.

| Tool | Purpose |
|------|---------|
| Lynis | Quick audit and suggestions |
| OpenSCAP + SCAP Security Guide | CIS/STIG/PCI scanning and remediation |
| CIS-CAT | CIS's assessment tool |
| `ansible-lockdown`, `devsec.hardening` | Ansible roles that implement benchmarks |
| `rkhunter`, `chkrootkit` | Rootkit checks (limited value, can be bypassed) |
| `debsums`, `rpm -Va` | Package integrity |

### Hardening as code

Apply the baseline with configuration management (Ansible, Puppet, Salt) so every system is identical and drift is detected. Build **hardened images** (Packer, image builders) for cloud and virtual machines. Verify with scans in the pipeline.

## Common pitfalls

- **Applying a benchmark blindly.** It may break applications. Test, then document exceptions.
- **One-time hardening.** Systems drift. Use configuration management and re-scan regularly.
- **Hardening only the OS** while leaving applications, containers, and credentials weak.
- **Breaking remote access** (SSH, firewall, PAM changes). Always keep a second session.
- **`noexec` on `/tmp`** breaking updates or applications without testing.
- **Disabling IPv6 by sysctl** instead of managing it. Many tools expect it. Configure firewall rules for it instead.
- **Treating scan scores as security.** A 90% score does not make an exposed, unpatched service safe.
- **Hardening scripts copied from the internet** without understanding them.
- **No rollback plan** for kernel parameters, PAM, or SELinux changes.
- **Ignoring logging.** Hardening without detection means attacks are not noticed.

## Further reading

- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [OpenSCAP](https://www.open-scap.org)
- [ComplianceAsCode (SCAP Security Guide)](https://complianceascode.readthedocs.io)
- [Lynis](https://cisofy.com/lynis/)
- [DISA STIGs](https://public.cyber.mil/stigs/)
- [Red Hat: security hardening](https://docs.redhat.com)
- [Ubuntu Security Guide (USG)](https://ubuntu.com/security/certifications/docs/usg)
