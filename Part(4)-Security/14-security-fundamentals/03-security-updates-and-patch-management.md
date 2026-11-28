# Security Updates and Patch Management

## Overview

Most successful attacks use **known vulnerabilities for which a fix already exists**. Patch management is the process of learning about fixes, deciding how fast to apply them, testing, deploying, and verifying. For installation commands see [Package Verification, Dependencies, and Update Management](../../part-2-system-administration/07-package-management/06-verification-and-update-management.md). This page covers the **process and policy**.

## Key concepts

### Where to learn about vulnerabilities

| Source | Content |
|--------|---------|
| **Vendor advisories** | Ubuntu USN, Debian DSA/DLA, Red Hat RHSA, SUSE-SU, Rocky/Alma errata |
| **CVE** (Common Vulnerabilities and Exposures) | A unique ID per vulnerability (`CVE-2024-12345`) |
| **NVD** (National Vulnerability Database) | Details and **CVSS** scores |
| **CISA KEV catalog** | Vulnerabilities **known to be exploited in the wild** |
| **EPSS** | Probability that a CVE will be exploited |
| **Security mailing lists, upstream project announcements** | Early information |

### Vendor security trackers

| Family | Tracker |
|--------|---------|
| Debian | `security-tracker.debian.org` |
| Ubuntu | `ubuntu.com/security/cves` |
| RHEL-based | `access.redhat.com/security` |
| SUSE | `suse.com/security/cve` |

### Backporting (important)

Enterprise distributions keep an old version number and **backport** the fix. `openssl 1.1.1k` on RHEL 8 may be fully patched for a CVE that was "fixed in 1.1.1l" upstream. Scanners that compare version numbers only create false positives. Check the **vendor tracker** or the package **changelog**.

### Prioritization

Do not rank by CVSS score alone. Combine:

| Factor | Question |
|--------|----------|
| **Exploited now?** | In CISA KEV, or public exploit code? |
| **Exposure** | Reachable from the internet? Or only local? |
| **Asset value** | Holds sensitive data or critical function? |
| **Severity** | CVSS, and whether the conditions apply to your configuration |
| **Compensating controls** | Firewall, MAC, feature disabled |

### Example patch SLA (adapt to your policy)

| Priority | Examples | Target time |
|----------|----------|-------------|
| **Emergency** | Actively exploited, internet-facing | Hours to 2 days (mitigate first if needed) |
| **High** | Critical/High with remote exploit | 7 days |
| **Medium** | Typical security updates | 30 days |
| **Low** | Minor or local-only | Next regular maintenance (up to 90 days) |

### Live patching and reboots

Kernel and core library updates need a **reboot** (or service restart) to take effect. **Live patching** applies critical kernel fixes without a reboot:

| Service | Provider |
|---------|----------|
| Canonical Livepatch (Ubuntu Pro) | Ubuntu |
| `kpatch` (RHEL subscription) | Red Hat |
| SUSE Live Patching | SUSE |
| KernelCare | Commercial, multiple distributions |

Live patching is a **bridge** until the next planned reboot, not a replacement for rebooting.

## Commands

### What needs patching?

```console
# apt update && apt list --upgradable 2>/dev/null | head       # Debian-based
$ pro security-status                                          # Ubuntu: summary (Ubuntu Pro client)
$ apt-get -s upgrade | grep -i '^Inst.*security'               # simulate; show security updates
$ debsecan --suite bookworm --only-fixed --format summary      # Debian: known CVEs for installed packages (package debsecan)

# dnf updateinfo summary                                       # RHEL-based
# dnf updateinfo list security
# dnf updateinfo info --cve CVE-2024-12345
# dnf check-update --security

# zypper list-patches --category security                      # SUSE-based
# zypper patch-check
```

### Apply security updates only

```console
# apt-get install --only-upgrade PACKAGE                        # one package
# unattended-upgrade -d                                          # Debian-based, as configured (security only by default)
# dnf upgrade --security                                         # RHEL-based
# dnf upgrade --cve CVE-2024-12345                               # a specific CVE
# zypper patch --category security                               # SUSE-based
```

### Does a package contain the fix?

```console
$ apt changelog openssl | grep -i 'CVE-2024-12345'              # Debian-based
$ rpm -q --changelog openssl | grep -i 'CVE-2024-12345'         # RHEL-based / SUSE-based
```

### Reboot and restart needs

```console
$ ls /var/run/reboot-required && cat /var/run/reboot-required.pkgs     # Debian-based
$ needrestart -r l                                                      # Debian-based: services needing restart
# dnf needs-restarting -r                                               # RHEL-based: reboot needed? (exit 1 = yes)
# dnf needs-restarting -s                                               # services to restart
# zypper ps -s                                                          # SUSE-based
$ uname -r; ls /boot/vmlinuz-*                                          # running vs installed kernels
```

### Automatic updates (summary)

| Family | Tool and key settings |
|--------|----------------------|
| Debian-based | `unattended-upgrades`: `/etc/apt/apt.conf.d/50unattended-upgrades` (`Unattended-Upgrade::Allowed-Origins`, `Mail`, `Automatic-Reboot`), `20auto-upgrades` |
| RHEL-based | `dnf-automatic`: `/etc/dnf/automatic.conf` (`upgrade_type = security`, `apply_updates = yes`), `dnf-automatic.timer` |
| SUSE-based | `zypper patch` from a timer, or SUSE Manager / `transactional-update` |

### A patch management process

1. **Inventory:** know every host, OS version, installed packages, and owner (CMDB, configuration management).
2. **Track** advisories for your distributions and key software.
3. **Assess** and **prioritize** (exposure, exploitation, asset value).
4. **Test** on staging systems that mirror production.
5. **Schedule** a maintenance window. Notify stakeholders.
6. **Back up or snapshot** before patching.
7. **Deploy** in stages: canary hosts → a small group → everything. Patch **one node at a time** in clusters.
8. **Restart** services and **reboot** where needed.
9. **Verify**: service health checks, `systemctl --failed`, logs, vulnerability re-scan.
10. **Report** and **document** exceptions, with an expiry date and compensating controls.
11. **Improve**: shorten time to patch, automate.

### Other things to patch

| Item | How |
|------|-----|
| **Containers and images** | Rebuild images regularly from patched bases. Scan them (Trivy, Grype). Redeploy |
| **Third-party repositories and manually installed software** | Track versions yourself. Subscribe to project announcements |
| **Language packages** (pip, npm, Maven, Go modules) | Dependency scanners (`pip-audit`, `npm audit`, Dependabot, Renovate) |
| **Firmware and BIOS** | `fwupdmgr` (LVFS), vendor tools |
| **Network devices, hypervisors, storage** | Vendor processes |
| **End-of-life systems** | Migrate. If impossible, isolate heavily and use extended support |

```console
$ fwupdmgr refresh && fwupdmgr get-updates
```

## Common pitfalls

- **"If it works, don't touch it."** Unpatched systems are the most exploited.
- **Applying updates but not restarting services or rebooting.** The vulnerable code stays in memory. Check `needrestart` / `needs-restarting`.
- **Trusting scanner version comparisons** on backported packages. Verify against the vendor tracker.
- **Ignoring non-OS software** (containers, language dependencies, appliances).
- **Patching everything at once** without staging, then having no rollback. Use canaries and snapshots.
- **No ownership.** Nobody is responsible for patching a system. Assign owners.
- **Enabling third-party repositories** that replace system packages (see [Repositories](../../part-2-system-administration/07-package-management/05-repositories.md)).
- **Holding packages forever.** Track every hold with a reason and an expiry.
- **Auto-reboot in production without coordination** can cause outages. Use rolling reboots (for example, `reboot-required` plus orchestration, or `kured` in Kubernetes).
- **Missing verification.** After patching, confirm the fixed version is running (`rpm -q`, `dpkg -l`, `uname -r`, process start time).

## Further reading

- [NIST SP 800-40: Guide to Enterprise Patch Management](https://csrc.nist.gov/publications/detail/sp/800-40/rev-4/final)
- [CISA Known Exploited Vulnerabilities catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [FIRST: EPSS](https://www.first.org/epss/)
- [Debian security tracker](https://security-tracker.debian.org)
- [Ubuntu Security](https://ubuntu.com/security)
- [Red Hat Customer Portal: security](https://access.redhat.com/security)
