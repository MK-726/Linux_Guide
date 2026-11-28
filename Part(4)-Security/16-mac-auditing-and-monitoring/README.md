# Chapter 16: Mandatory Access Control, Auditing & Security Monitoring

| # | Topic | Summary |
|---|-------|---------|
| 1 | [DAC and MAC Concepts](01-dac-and-mac-concepts.md) | Why permissions are not enough |
| 2 | [SELinux: Modes and Contexts](02-selinux-modes-and-contexts.md) | Enforcing, permissive, labels |
| 3 | [SELinux: Policies and Booleans](03-selinux-policies-and-booleans.md) | Booleans, ports, file contexts, custom modules |
| 4 | [SELinux: Troubleshooting](04-selinux-troubleshooting.md) | Read denials, fix the right thing |
| 5 | [AppArmor](05-apparmor.md) | Path-based confinement (Ubuntu, SUSE) |
| 6 | [Linux Auditing and auditd](06-linux-auditing-and-auditd.md) | Record security-relevant events |
| 7 | [File Integrity Monitoring](07-file-integrity-monitoring.md) | AIDE and detecting unauthorized changes |
| 8 | [Security Logs and Event Analysis](08-security-logs-and-event-analysis.md) | Find attacks in logs |
| 9 | [Cryptography Fundamentals](09-cryptography-fundamentals.md) | Hashes, encryption, signatures, GPG, OpenSSL |
| 10 | [TLS and Certificate Management](10-tls-and-certificate-management.md) | CSRs, CAs, ACME, trust stores, expiry |
| 11 | [Secrets Management](11-secrets-management.md) | Store and deliver credentials safely |
| 12 | [Vulnerability Assessment Fundamentals](12-vulnerability-assessment-fundamentals.md) | Scanning, CVSS, prioritization |

> [!WARNING]
> **Do not disable SELinux or AppArmor to fix a problem.** Learn to read the denial and adjust labels, booleans, or policy. Disabling removes a key security layer and often hides the real mistake.

**Previous:** [Chapter 15](../15-authentication-and-identity/README.md) | **Next part:** Part 5: Shell Scripting, Programming & Automation (planned)
