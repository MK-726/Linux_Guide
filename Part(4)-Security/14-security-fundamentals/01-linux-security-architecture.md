# Linux Security Architecture

## Overview

Linux security is built from **many independent layers**. No single mechanism is enough. The idea is **defense in depth**: if one layer fails, another still limits the damage. This page gives the map. The following pages and chapters cover each layer in detail.

## Key concepts

### Security goals (the CIA triad)

| Goal | Meaning | Example threat |
|------|---------|----------------|
| **Confidentiality** | Only authorized parties can read data | Stolen database, leaked secrets |
| **Integrity** | Data and systems are not changed without authorization | Tampered binary, altered logs |
| **Availability** | Systems work when needed | DDoS, ransomware, disk full |

### The layers

```
┌──────────────────────────────────────────────────────────────┐
│ People and process: policy, training, change control          │
├──────────────────────────────────────────────────────────────┤
│ Network: firewalls, segmentation, TLS, VPN                    │
├──────────────────────────────────────────────────────────────┤
│ Applications: updates, secure configuration, sandboxing       │
├──────────────────────────────────────────────────────────────┤
│ Identity: authentication (PAM, SSH keys, MFA), sudo           │
├──────────────────────────────────────────────────────────────┤
│ Access control: DAC (permissions, ACLs), MAC (SELinux/AppArmor)│
├──────────────────────────────────────────────────────────────┤
│ Isolation: users, namespaces, cgroups, seccomp, capabilities  │
├──────────────────────────────────────────────────────────────┤
│ Kernel hardening: ASLR, NX, lockdown, restricted interfaces   │
├──────────────────────────────────────────────────────────────┤
│ Boot and hardware: UEFI Secure Boot, TPM, disk encryption     │
├──────────────────────────────────────────────────────────────┤
│ Physical security                                             │
└──────────────────────────────────────────────────────────────┘
```

### The core security model

| Element | Role |
|---------|------|
| **Users and groups** | Identity of a process. Every process runs as a user |
| **Files and permissions** | **DAC** (Discretionary Access Control): the owner decides who can access (see [File Ownership and Permissions](../../part-1-foundations/04-users-groups-and-permissions/04-ownership-and-permissions.md)) |
| **root (UID 0)** | Bypasses most checks. The main target of attackers |
| **Kernel / user space split** | Programs cannot touch hardware or other processes' memory directly (see [Kernel Architecture](../../part-2-system-administration/05-system-internals/01-kernel-architecture-and-system-calls.md)) |
| **Capabilities** | Break root's power into about 40 smaller privileges |
| **Namespaces** | Limit **what a process can see** (processes, mounts, network, users) |
| **cgroups** | Limit **how much** a group of processes can use |
| **seccomp** | Filter which **system calls** a process may make |
| **LSM (Linux Security Modules)** | Framework for **mandatory** access control: SELinux, AppArmor, and others |
| **Audit subsystem** | Records security-relevant events |

### Mandatory vs. discretionary access control

- **DAC:** the file owner (or root) decides. A compromised user process can do whatever that user can.
- **MAC:** a **system-wide policy** decides, and even root-owned processes are confined by it. SELinux and AppArmor confine a service to what it needs (Chapter 16).

### Kernel hardening features

| Feature | What it does |
|---------|--------------|
| **ASLR** (`kernel.randomize_va_space=2`) | Randomizes memory layout, making exploits harder |
| **NX / DEP** | Marks data memory non-executable |
| **Stack protector, hardened usercopy, KASLR** | Compile-time and boot-time protections |
| **SMEP / SMAP** | CPU features that block the kernel from executing/accessing user memory |
| **Kernel lockdown** | Restricts changes to the running kernel, even for root (works with Secure Boot) |
| **Yama** (`kernel.yama.ptrace_scope`) | Limits which processes can `ptrace` others |
| **Restricted interfaces** | `kernel.dmesg_restrict`, `kernel.kptr_restrict`, `kernel.unprivileged_bpf_disabled` |
| **Protected links** | `fs.protected_hardlinks`, `fs.protected_symlinks`, `fs.protected_fifos`, `fs.protected_regular` limit common `/tmp` attacks |
| **Landlock** | Unprivileged self-sandboxing of filesystem access (newer kernels) |

### Trust boundaries

A **trust boundary** is a place where data or control crosses from a less trusted to a more trusted domain: network to host, user to kernel, container to host, user process to root service. Most vulnerabilities occur at boundaries. Examples: a web request reaching an application, a normal user running a SUID program, a container escaping to the host.

## Commands

Inspect the protections on your system:

```console
$ sysctl kernel.randomize_va_space kernel.kptr_restrict kernel.dmesg_restrict kernel.yama.ptrace_scope
kernel.randomize_va_space = 2
kernel.kptr_restrict = 1
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1

$ cat /sys/kernel/security/lsm                 # active security modules
lockdown,capability,landlock,yama,apparmor,ima,evm

$ cat /sys/kernel/security/lockdown            # kernel lockdown mode
none [integrity] confidentiality

$ getenforce 2>/dev/null; aa-status 2>/dev/null | head -3     # SELinux mode / AppArmor status
$ mokutil --sb-state                           # UEFI Secure Boot
$ grep -i -E 'nx|smep|smap' /proc/cpuinfo | head -1 | tr ' ' '\n' | grep -E '^(nx|smep|smap)$'
$ cat /proc/sys/kernel/seccomp/actions_avail 2>/dev/null
$ grep -E 'Seccomp|NoNewPrivs|CapEff' /proc/self/status
Seccomp:        0
NoNewPrivs:     0
CapEff: 0000000000000000
```

`Seccomp: 2` in a process's status means a seccomp filter is active.

### How do I see what a process is allowed?

```console
$ ps -eo pid,user,comm,label | head -5          # security labels (SELinux context)
$ cat /proc/PID/attr/current                     # SELinux/AppArmor label
$ grep -E 'Seccomp|NoNewPrivs|Cap' /proc/PID/status
$ systemd-analyze security nginx.service         # sandboxing score of a service
```

## Common pitfalls

- **Relying on one control.** A firewall alone, or file permissions alone, is not a security design.
- **Disabling SELinux or AppArmor "to make it work".** Fix the policy instead. Disabling removes a key layer (Chapter 16).
- **Treating root inside a container as harmless.** A privileged container or a kernel bug can give host access.
- **Thinking Linux is secure by default.** Defaults vary. Services, accounts, and permissions must still be reviewed.
- **Security through obscurity.** A changed port or hidden path is not a control.
- **Ignoring physical access.** Anyone with console and no disk encryption can reset passwords (see [Boot Troubleshooting](../../part-2-system-administration/06-boot-grub-systemd/07-boot-troubleshooting-and-recovery.md)).
- **Forgetting that tools are not equal across distributions.** RHEL-based systems default to SELinux, Ubuntu and SUSE to AppArmor.

## Further reading

- [Linux kernel: security documentation](https://www.kernel.org/doc/html/latest/security/index.html)
- [`capabilities(7)`](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [`namespaces(7)`](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [`seccomp(2)`](https://man7.org/linux/man-pages/man2/seccomp.2.html)
- [NIST SP 800-123: Guide to General Server Security](https://csrc.nist.gov/publications/detail/sp/800-123/final)
