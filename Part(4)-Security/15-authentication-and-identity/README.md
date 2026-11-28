# Chapter 15: Authentication, Authorization & Identity Management

| # | Topic | Summary |
|---|-------|---------|
| 1 | [Authentication vs. Authorization](01-authentication-vs-authorization.md) | Who you are vs. what you may do |
| 2 | [PAM: Pluggable Authentication Modules](02-pam.md) | The login and authentication framework |
| 3 | [NSS: Name Service Switch](03-nss.md) | Where Linux looks up users, groups, and hosts |
| 4 | [Password Policies and Aging](04-password-policies-and-aging.md) | Strength, lockout, hashing, expiry |
| 5 | [sudoers Best Practices](05-sudoers-best-practices.md) | Safe, auditable privilege delegation |
| 6 | [SSH Authentication and Key Management](06-ssh-authentication-and-key-management.md) | Key lifecycle, certificates, central keys |
| 7 | [Multi-Factor Authentication Concepts](07-multi-factor-authentication-concepts.md) | TOTP, FIDO2, PAM integration |
| 8 | [LDAP Fundamentals](08-ldap-fundamentals.md) | Directory structure, queries, clients |
| 9 | [Active Directory Integration](09-active-directory-integration.md) | Join Linux to AD with `realmd` and SSSD |
| 10 | [SSSD and Centralized Identity Management](10-sssd-and-centralized-identity-management.md) | One identity source for many servers |

> [!WARNING]
> Changes to PAM, NSS, SSSD, and `sudo` can **lock everyone out**, including administrators. Always keep a root shell open in a second session, have console access, and test changes with a separate login before you close anything.

**Previous:** [Chapter 14](../14-security-fundamentals/README.md) | **Next:** [Chapter 16](../16-mac-auditing-and-monitoring/README.md)
