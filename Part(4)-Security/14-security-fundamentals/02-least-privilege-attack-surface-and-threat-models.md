# Least Privilege, Attack Surface, and Threat Models

## Overview

Good security starts with three questions: *What am I protecting? Who or what might attack it, and how? How do I give every user and program only the access it needs?* The answers are the **threat model**, the **attack surface**, and the **principle of least privilege**.

## Key concepts

### Principle of least privilege

Every user, process, and system should have **only the permissions it needs, for only as long as it needs them**.

| Area | Least-privilege practice |
|------|--------------------------|
| Users | Normal accounts for daily work. `sudo` for specific commands, not a root shell by default |
| Services | A dedicated, unprivileged **service account** per service. No login shell |
| Files | Owner and group set so only the service and administrators can read secrets (`640`, `600`) |
| Network | Firewall default deny. Each service reachable only from the sources that need it |
| Capabilities | Grant single capabilities (for example `CAP_NET_BIND_SERVICE`) instead of root |
| Systemd | Sandboxing options: `ProtectSystem`, `NoNewPrivileges`, `PrivateTmp` (see [Capabilities and Service Isolation](06-capabilities-and-service-isolation.md)) |
| Automation | Separate keys and accounts per task, with `command=` restrictions |
| Time | Temporary access that expires (short-lived credentials, `sudo` timeouts, break-glass accounts) |
| Cloud | Roles with narrow permissions, not administrator rights for all |

Related principles:

| Principle | Meaning |
|-----------|---------|
| **Defense in depth** | Multiple independent layers |
| **Fail secure** | When something breaks, deny access rather than allow |
| **Separation of duties** | No single person controls an entire critical process |
| **Minimize trust** | Verify identity and integrity. Do not assume the internal network is safe |
| **Economy of mechanism** | Simple designs are easier to secure |

### Attack surface

The **attack surface** is everything an attacker can reach or interact with: open ports, running services, installed software, user accounts, exposed APIs, SUID programs, scheduled jobs, and the people who administer the system. **Reduce it**, and there is less to attack and less to patch.

Typical entry points on a Linux server:

| Entry point | Examples |
|-------------|----------|
| **Network services** | SSH, web servers, databases, management interfaces |
| **Local privilege escalation paths** | SUID binaries, writable service files, weak `sudo` rules, kernel bugs |
| **Credentials** | Weak or reused passwords, leaked keys, secrets in scripts or repositories |
| **Supply chain** | Compromised packages, repositories, container images, build pipelines |
| **Configuration** | Default settings, debug interfaces, open file shares |
| **Physical and console** | USB, serial console, recovery mode |
| **People** | Phishing, social engineering, insider misuse |

### Threat modeling

A **threat model** is a structured answer to:

1. **What are we protecting?** (assets: customer data, credentials, uptime, reputation)
2. **Who might attack?** (actors: opportunistic bots, criminals, insiders, competitors, state-level attackers)
3. **How could they?** (attack paths)
4. **What do we do about it?** (controls, accepted risks)
5. **Did it work?** (testing, monitoring, review)

A common memory aid for finding threats is **STRIDE**:

| Letter | Threat | Property violated | Example |
|--------|--------|-------------------|---------|
| **S** | Spoofing | Authentication | Stolen SSH key used to log in |
| **T** | Tampering | Integrity | Modified configuration or binary |
| **R** | Repudiation | Non-repudiation | No logs, so actions cannot be traced |
| **I** | Information disclosure | Confidentiality | World-readable backup with secrets |
| **D** | Denial of service | Availability | Resource exhaustion, ransomware |
| **E** | Elevation of privilege | Authorization | User gains root through a SUID bug |

**Risk** is often estimated as **likelihood × impact**. Spend effort where both are high.

### The attack chain

Attackers usually follow steps: **reconnaissance → initial access → execution → persistence → privilege escalation → lateral movement → data theft or damage**. Defenders aim to block early steps, detect the middle ones, and limit the last ones. (The MITRE ATT&CK framework catalogs these techniques.)

### Example mini threat model: a public web server

| Asset | Threat | Control |
|-------|--------|---------|
| Application data | SQL injection, stolen database | Parameterized queries, database on private network, least-privilege DB user, WAF, backups |
| Server integrity | Vulnerable web server software | Patch management, minimal packages, automatic security updates |
| Admin access | Brute-force and stolen SSH keys | Key-only SSH, MFA, source restriction, `fail2ban`, bastion |
| Secrets | Keys in the repository | Secrets manager, scanning, rotation |
| Availability | DDoS, resource exhaustion | CDN or rate limiting, resource limits (cgroups), monitoring |
| Detection | Attack goes unnoticed | Central logging, `auditd`, alerts, file integrity monitoring |

## Commands

Measure the attack surface of a host:

```console
$ ss -tulpn                                          # exposed network services
$ systemctl list-unit-files --state=enabled --type=service     # what starts at boot
$ systemctl list-timers --all                        # scheduled tasks
$ dpkg -l | wc -l            # Debian-based: installed packages (fewer is better)
$ rpm -qa | wc -l            # RHEL-based / SUSE-based
$ find / -xdev -perm -4000 -type f 2>/dev/null       # SUID programs
$ find / -xdev -perm -2000 -type f 2>/dev/null       # SGID programs
$ getcap -r / 2>/dev/null                            # files with capabilities
$ awk -F: '$3==0 {print $1}' /etc/passwd             # UID 0 accounts (should be only root)
$ awk -F: '$2=="" {print $1}' /etc/shadow            # accounts with no password (needs root)
$ grep -v -E 'nologin|false' /etc/passwd             # accounts with login shells
$ sudo -l -U alice                                   # what alice may run as root
$ lastlog | grep -v 'Never'                          # which accounts are used
$ find /etc/cron* /var/spool/cron -type f 2>/dev/null | head
```

Review services for exposure and sandboxing:

```console
$ systemd-analyze security --no-pager | head -20     # exposure score for each service
```

## A practical process

1. **Inventory** systems, data, and owners.
2. **Model threats** for each important system (one page is enough).
3. **Reduce the attack surface**: remove packages, disable services, close ports.
4. **Apply least privilege**: accounts, `sudo`, file permissions, capabilities, sandboxing.
5. **Add layers**: firewall, MAC, auditing, logging.
6. **Monitor and test**: vulnerability scans, log review, incident drills.
7. **Review regularly**: after each change, and at fixed intervals.

## Common pitfalls

- **Giving broad access "for now."** Temporary exceptions become permanent. Set expiry dates.
- **Shared accounts** (one `admin` login for everyone). No accountability. Use personal accounts and `sudo`.
- **Over-trusting the internal network.** Once attackers are inside, flat networks let them move freely. Segment and authenticate internal traffic too.
- **Hardening without a threat model.** Effort goes to the wrong places. Protect the highest-value assets first.
- **Counting installed software only on servers** and ignoring build tools, compilers, and package managers. They help an attacker after a break-in. Remove them from production images when possible.
- **`NOPASSWD: ALL` in sudoers** for convenience.
- **Forgetting non-human accounts** (service accounts, CI tokens, cloud roles), which often have excessive rights and never expire.
- **Never revisiting the model.** New services, new integrations, and new staff change the risk.

## Further reading

- [OWASP Threat Modeling](https://owasp.org/www-community/Threat_Modeling)
- [Microsoft STRIDE overview](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats)
- [MITRE ATT&CK](https://attack.mitre.org)
- [NIST SP 800-53: security and privacy controls](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [Saltzer and Schroeder: design principles](https://www.cs.virginia.edu/~evans/cs551/saltzer/)
