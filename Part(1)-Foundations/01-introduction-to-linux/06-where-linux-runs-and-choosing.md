# Where Linux Runs and Choosing a Distribution

## Overview

Linux runs on everything from tiny devices to supercomputers. This page shows the main environments and gives a simple method for choosing a distribution.

## Key concepts

### Where Linux is used

| Environment | Why Linux is used |
|-------------|-------------------|
| **Servers** | Stable, secure, scriptable, no per-copy license fee for most distributions |
| **Cloud infrastructure** | Most cloud virtual machines run Linux. Cloud platforms offer ready-made Linux images. |
| **Containers** | Containers use Linux kernel features (namespaces and cgroups). Container hosts are almost always Linux. |
| **Desktops and laptops** | Used by developers and administrators; many desktop environments are available |
| **Embedded and IoT** | Routers, TVs, and industrial devices |
| **Mobile** | Android uses the Linux kernel |
| **Supercomputers** | Nearly all of the world's fastest systems run Linux |

### Choosing a distribution

Ask these questions in order:

1. **Does your organization already standardize on a distribution?** If yes, use it. Consistency saves time.
2. **Do you need vendor support or certification?** Choose RHEL, SLES, or Ubuntu Pro. Some software vendors certify only specific distributions.
3. **How long must the system run without a major upgrade?** Choose an LTS or enterprise release.
4. **Do you need the newest software?** Consider Fedora or a rolling release, but not for critical production servers.
5. **What skills does your team have?** Pick the family your team knows.
6. **What is the workload?** Containers may use a tiny base such as Alpine. Cloud workloads may use the cloud vendor's image.

### Quick guide

| Need | Good choices |
|------|-------------|
| General-purpose server, large community | Ubuntu LTS, Debian stable |
| Enterprise with vendor support | RHEL, SLES |
| Free RHEL-compatible server | Rocky Linux, AlmaLinux |
| Learning enterprise skills (RHCSA) | Rocky Linux, AlmaLinux, Fedora, RHEL Developer subscription |
| Newest software for desktop or development | Fedora, Ubuntu (latest release), openSUSE Tumbleweed |
| Minimal container image | Alpine, Debian slim, Ubuntu minimal |
| AWS workloads | Amazon Linux (also Ubuntu, RHEL) |

## Commands

Inspect what environment your Linux system runs in:

```console
$ systemd-detect-virt
kvm

$ hostnamectl | grep -E 'Chassis|Virtualization|Operating System'
       Chassis: vm
Virtualization: kvm
Operating System: Ubuntu 24.04 LTS
```

| `systemd-detect-virt` result | Meaning |
|------------------------------|---------|
| `none` | Physical machine |
| `kvm`, `vmware`, `oracle` | Virtual machine (hypervisor type) |
| `docker`, `podman`, `lxc` | Container |

## Common pitfalls

- **Choosing by popularity only.** The "best" distribution is the one that fits your support, lifecycle, and skill requirements.
- **Using a desktop-oriented release for servers.** Use server or minimal installs without a graphical desktop.
- **Ignoring migration cost.** Moving a fleet from one family to another needs retraining and tooling changes.

## Further reading

- [DistroWatch](https://distrowatch.com)
- [Cloud provider image documentation](https://docs.aws.amazon.com/linux/)
