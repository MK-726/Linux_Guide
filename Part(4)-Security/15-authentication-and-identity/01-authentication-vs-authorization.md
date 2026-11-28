# Authentication vs. Authorization

## Overview

Two questions control access to any system:

1. **Authentication (AuthN):** *Who are you?* Prove your identity.
2. **Authorization (AuthZ):** *What are you allowed to do?* Decide access after identity is known.

A third, **accounting** (auditing), records *what you did*. Together they are called **AAA**. Mixing them up leads to design mistakes: a strong login does not help if every user is authorized to do everything.

## Key concepts

### Authentication

You prove identity with one or more **factors**:

| Factor | Meaning | Examples |
|--------|---------|----------|
| **Something you know** | Knowledge | Password, passphrase, PIN |
| **Something you have** | Possession | SSH private key, hardware security key (FIDO2), smart card, phone with an authenticator app |
| **Something you are** | Inherence | Fingerprint, face (mostly on workstations) |
| (Somewhere you are / when) | Context | Source network, time of day (a supporting signal, not a factor by itself) |

**Multi-factor authentication (MFA)** needs two or more **different** factors. A password plus a security question is not MFA (both are knowledge).

Authentication mechanisms on Linux:

| Mechanism | Used by |
|-----------|---------|
| Local passwords (`/etc/shadow`) | Console, SSH, `su`, `sudo` |
| SSH public keys | SSH |
| **Kerberos** tickets | Single sign-on in domains (AD, FreeIPA) |
| LDAP bind | Directory-based passwords |
| Certificates (TLS client certs, smart cards) | VPN, web, SSH certificates |
| Tokens and API keys | Services, automation |
| OTP (TOTP/HOTP) | Second factor |
| FIDO2/WebAuthn | Phishing-resistant second factor or passwordless |

### Authorization

After authentication, the system decides what the identity may do. On Linux this happens at **many layers**:

| Layer | Mechanism |
|-------|-----------|
| Files | Owner/group/mode, ACLs ([Ownership and Permissions](../../part-1-foundations/04-users-groups-and-permissions/04-ownership-and-permissions.md)) |
| Privilege | `sudo` rules ([sudoers Best Practices](05-sudoers-best-practices.md)), capabilities, `su` restrictions |
| Login rights | PAM `account` modules: `pam_access`, `pam_time`, `pam_nologin`, SSSD access control, `AllowGroups` in sshd |
| Desktop and system actions | **polkit** rules (`pkexec`, systemd actions) |
| Process confinement | SELinux/AppArmor (Chapter 16) |
| Network | Firewall rules |
| Applications | Roles in the application (RBAC), database grants |
| Central policy | FreeIPA **HBAC** (host-based access control), AD group policy |

### How they work together in an SSH login

```
1. Client connects; server proves its identity (host key).            ← server authentication
2. User presents a key or password (+ MFA).                           ← authentication (sshd + PAM)
3. PAM "account" stage checks: account locked or expired? allowed to log in here?   ← authorization
4. sshd checks AllowUsers/AllowGroups, Match rules.                    ← authorization
5. Session starts as the user. File permissions, sudo rules, SELinux apply.         ← authorization
6. Everything is logged (sshd, auth log, auditd).                      ← accounting
```

### Identity vs. access

| Term | Meaning |
|------|---------|
| **Identity** | A person or service with a unique account (and attributes such as groups) |
| **Credential** | What proves the identity (password, key, certificate) |
| **Entitlement / permission** | What the identity may do |
| **Role / group** | A collection of entitlements, assigned to many identities |
| **Joiner-mover-leaver** | The lifecycle: create access when someone joins, adjust when they change roles, **remove it when they leave** |
| **Single sign-on (SSO)** | One login gives access to many systems (Kerberos, SAML, OIDC) |
| **Identity provider (IdP)** | The system that stores identities and authenticates them (AD, FreeIPA, Okta, Entra ID, Keycloak) |

### Design principles

1. **Individual accounts.** Never share accounts.
2. **Strong authentication** for privileged access (MFA, keys).
3. **Centralize identity**, so joiners, movers, and leavers are handled in one place (see [SSSD and Centralized Identity Management](10-sssd-and-centralized-identity-management.md)).
4. **Authorize by role/group**, not by individual rules on each server.
5. **Least privilege** for every account, including service accounts.
6. **Log everything** relevant to authentication and privilege use.
7. **Review access regularly** and remove what is not needed.
8. **Plan emergency access** (a protected break-glass account) so an identity outage does not lock you out.

## Commands

### See identity and authorization facts for an account

```console
$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),1005(developers)
$ getent passwd alice
$ getent group developers
$ sudo -l -U alice                         # what alice may run with sudo
$ passwd -S alice                          # password state (P = usable, L = locked, NP = none)
$ chage -l alice                           # expiry and aging
$ lastlog -u alice                         # last login
$ last -n 5 alice
$ faillock --user alice                    # failed login attempts (RHEL-based; pam_faillock)
$ sshd -T -C user=alice,host=server01,addr=192.0.2.9 | grep -i -E 'allowusers|allowgroups|authenticationmethods|permitrootlogin'
$ pkaction | head                          # polkit actions
$ pkcheck --action-id org.freedesktop.systemd1.manage-units --process $$ ; echo $?
```

### Find where authentication happens for a service

```console
$ ls /etc/pam.d/                           # each PAM-aware service has a file
$ cat /etc/pam.d/sshd
$ grep -E '^(passwd|group|shadow|sudoers):' /etc/nsswitch.conf
```

## Common pitfalls

- **Treating authentication as the whole solution.** Everybody who can log in can then do everything, because there are no authorization limits.
- **Shared accounts and shared keys.** No accountability and impossible to revoke for one person.
- **Over-privileged service accounts** that never expire.
- **No leaver process.** Accounts and SSH keys of former staff remain valid.
- **Mixing up factors.** Two passwords are not MFA.
- **SMS as the only second factor.** Weak against SIM swapping and phishing. Prefer FIDO2 or app-based TOTP.
- **Group sprawl.** Many overlapping groups nobody understands. Review membership regularly.
- **Authorizing by hostname or IP only** (for example, NFS `sec=sys`, `from=` as the sole control).
- **No break-glass access** to central identity failures.
- **Ignoring non-interactive authentication** (cron jobs, CI/CD tokens, API keys). They need the same lifecycle management.

## Further reading

- [NIST SP 800-63B: Digital Identity Guidelines (Authentication)](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [NIST SP 800-53: AC (Access Control) and IA (Identification and Authentication) families](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [`polkit(8)`](https://man7.org/linux/man-pages/man8/polkit.8.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
