# Multi-Factor Authentication Concepts

## Overview

**Multi-factor authentication (MFA)** requires proof from **two or more different kinds** of factors (something you know, something you have, something you are). If an attacker steals one factor (a password), they still cannot log in. MFA is one of the most effective defenses against credential theft and should protect all privileged and remote access.

## Key concepts

### Methods compared

| Method | Factor | Phishing-resistant? | Notes |
|--------|--------|---------------------|-------|
| Password | Knowledge | No | Baseline |
| **SMS / voice code** | Possession (weak) | No | SIM swapping, interception. Avoid as the only second factor |
| **TOTP** (authenticator app, RFC 6238) | Possession | **No** (a phished code can be replayed within its 30 seconds) | Easy, free, offline. Good baseline |
| **Push notification** | Possession | Partly | "MFA fatigue" attacks (users approve by mistake). Use number matching |
| **FIDO2 / WebAuthn / U2F security key** | Possession (+ PIN/biometric) | **Yes** | Best. The key checks the site or host identity |
| **SSH key with a hardware token** (`ed25519-sk`) | Possession (+ touch) | Yes | For SSH |
| **Smart card / PIV / certificate** | Possession + PIN | Yes | Common in government and enterprise |
| **Biometrics** | Inherence | Depends | Mostly on workstations, unlocking a local key |
| **Hardware OTP token** | Possession | No | Dedicated device |
| **SSH key + passphrase** | Possession + knowledge | Partial | Real two factors **only if the key is not copyable** (hardware key). A file-based key is copyable, which weakens the claim |

### TOTP in short

The server and the authenticator app share a **secret** (shown as a QR code at enrollment). Both compute a 6-digit code from the secret and the current time (30-second steps). **Clocks must be in sync** (see [NTP and chrony](../../part-2-system-administration/10-logging-time-and-scheduling/04-time-zones-ntp-and-chrony.md)).

### Where to apply MFA on a Linux host

| Entry point | How |
|-------------|-----|
| **SSH** | PAM (`pam_google_authenticator`, `pam_u2f`, RADIUS), FIDO2 `sk` keys, or an access platform |
| **Console and local logins** | PAM `login`, `gdm`, `lightdm` |
| **`sudo`** | PAM `/etc/pam.d/sudo` (a second factor for privilege escalation) |
| **VPN** | OpenVPN, WireGuard front ends, IPsec with RADIUS or SAML |
| **Web applications and bastions** | SSO with MFA at the identity provider |
| **Central identity** | FreeIPA OTP, AD/Entra ID with MFA, RADIUS, privacyIDEA |

## Commands and examples

### TOTP for SSH using Google Authenticator PAM

Works with any TOTP app (Aegis, FreeOTP, Authy, Google Authenticator, a password manager).

**Install**

```console
# apt install libpam-google-authenticator       # Debian-based
# dnf install google-authenticator              # RHEL-based (EPEL)
```

**Enroll each user** (run as that user):

```console
$ google-authenticator -t -d -f -r 3 -R 30 -w 3
# prints a QR code, a secret key, and emergency scratch codes
```

Store the **scratch codes** safely. The secret is saved in `~/.google_authenticator` (mode 0400).

**Configure PAM** (`/etc/pam.d/sshd`):

```
# Debian-based: comment out the password prompt if keys are required
# @include common-auth
auth required pam_google_authenticator.so nullok
```

`nullok` lets users who have **not enrolled yet** log in without a code. **Remove it** after everyone has enrolled. Without it, users with no secret are locked out.

**Configure sshd** (`/etc/ssh/sshd_config.d/40-mfa.conf`):

```
UsePAM yes
KbdInteractiveAuthentication yes
PubkeyAuthentication yes
PasswordAuthentication no
AuthenticationMethods publickey,keyboard-interactive
```

This means: the user must present a valid **SSH key** (first factor) **and** answer the **verification code** prompt (second factor).

For **password + TOTP** instead of keys, use `AuthenticationMethods keyboard-interactive` and keep `common-auth` (password) before `pam_google_authenticator`.

```console
# sshd -t && systemctl reload sshd
# open a SECOND terminal and test login BEFORE closing the first
```

### FIDO2 / U2F with pam_u2f (local, sudo, and some SSH uses)

```console
# apt install libpam-u2f pamu2fcfg            # or: dnf install pam-u2f pamu2fcfg
$ mkdir -p ~/.config/Yubico
$ pamu2fcfg > ~/.config/Yubico/u2f_keys       # touch the key when it blinks
```

```
# /etc/pam.d/sudo (add before the other auth lines)
auth required pam_u2f.so cue
```

`sufficient` instead of `required` makes the key an alternative to a password, not an additional factor. Register a **second key** as a backup.

### SSH keys on hardware tokens (simplest strong SSH MFA)

```console
$ ssh-keygen -t ed25519-sk -O verify-required      # needs OpenSSH 8.2+ on both sides; touch and PIN required
```

On the server, no PAM changes are needed. Add the public key as usual. `-O verify-required` also demands the token PIN (`sshd_config`: `PubkeyAuthOptions verify-required` enforces it server-side).

### Centralized MFA

| Product | How it provides MFA |
|---------|--------------------|
| **FreeIPA** | OTP tokens per user and Kerberos authentication indicators. SSSD handles the prompts |
| **Active Directory / Entra ID** | MFA at the IdP. Linux logins via Kerberos plus an additional factor through supported agents or an access gateway |
| **RADIUS** (FreeRADIUS, Duo, privacyIDEA) | `pam_radius_auth` or `pam_duo`/`duo_unix` sends the second factor to a service |
| **SSH/access gateways** (Teleport, Boundary, cloud-provider tools) | SSO with MFA, short-lived certificates, session recording |
| **Smart cards** | SSSD supports smart card login (`pam_sss` with `pam_cert_auth`) |

### Test and troubleshoot

```console
$ ssh -vvv alice@server01 2>&1 | grep -i -E 'authentications that can continue|keyboard-interactive|publickey|verification'
# journalctl -u sshd -f
# tail -f /var/log/auth.log           # Debian-based; RHEL-based: /var/log/secure
# pamtester sshd alice authenticate
$ date -u; chronyc tracking | head -3                   # TOTP depends on correct time
```

| Problem | Likely cause |
|---------|--------------|
| "Verification code" always rejected | Clock drift on the server or phone. Check NTP. Window too small |
| No code prompt appears | `KbdInteractiveAuthentication no`, `UsePAM no`, or PAM line missing in `/etc/pam.d/sshd` |
| Password prompted in addition to key and code | `@include common-auth` still active in `/etc/pam.d/sshd` |
| User locked out | Not enrolled and no `nullok`, or lost phone. Use scratch codes or the admin recovery procedure |
| Works for SSH but not for `sudo` | `sudo` has its own PAM file (`/etc/pam.d/sudo`) |
| Automation breaks | Service accounts need a non-interactive method: separate restricted keys with `restrict`, `from=`, and `command=`, and `Match User` blocks that skip the second factor |

Skip MFA for specific users or networks with `Match`:

```
Match User deploy
    AuthenticationMethods publickey
Match Address 192.0.2.0/24
    AuthenticationMethods publickey
```

Use such exceptions only with other strong controls (restricted keys, network limits).

### Recovery planning

- **Scratch/recovery codes** stored securely.
- **At least two** registered tokens for important accounts.
- A documented **identity verification** process before resetting MFA.
- **Break-glass** access (console, vaulted account) that does not depend on the MFA system, with alerts when used.
- Test the recovery procedure.

## Common pitfalls

- **Enabling MFA and locking everyone out** (no `nullok` during rollout, wrong PAM order). Roll out in stages, with a second session open.
- **Time not synchronized** causing TOTP failures.
- **One MFA method for everything** with no fallback. Lost phones are common.
- **Treating SMS or push approvals as strong.** Prefer FIDO2. Use number matching for push.
- **Ignoring non-interactive access** (cron jobs, CI/CD, APIs). Give them restricted keys or tokens, not MFA bypass for everything.
- **Putting the TOTP secret on the same device and in the same password manager** as the password, which makes it a single factor in practice for some risks. Understand the trade-off.
- **Forgetting `sudo` and local console.** An attacker with a stolen session can still escalate. Consider MFA for `sudo` on critical servers.
- **Storing `~/.google_authenticator` with loose permissions or in backups unprotected.** It contains the secret.
- **Assuming MFA stops all attacks.** Session theft, malware, and real-time phishing proxies exist. Phishing-resistant methods and monitoring matter.
- **Weak enrollment process.** An attacker who can enroll their own device defeats MFA. Verify identity at enrollment.

## Further reading

- [NIST SP 800-63B: authenticator assurance levels](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 6238: TOTP](https://datatracker.ietf.org/doc/html/rfc6238)
- [FIDO Alliance](https://fidoalliance.org)
- [`pam_u2f`](https://developers.yubico.com/pam-u2f/)
- [google-authenticator-libpam](https://github.com/google/google-authenticator-libpam)
- [FreeIPA: OTP](https://freeipa.readthedocs.io/en/latest/workshop/11-two-factor-authentication.html)
