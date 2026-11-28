# Password Policies and Aging

## Overview

Passwords remain a main target of attacks (guessing, stuffing, phishing, theft). A good policy makes passwords hard to guess, limits guessing attempts, stores them safely, and uses MFA or keys where possible. Modern guidance (NIST SP 800-63B) favors **length and breached-password checks** over complex composition rules and forced periodic changes.

## Key concepts

### Current guidance (summary)

| Recommendation | Reason |
|----------------|--------|
| **Longer is better.** At least 12–15 characters, passphrases welcome | Length gives more strength than symbols |
| **Check against lists** of common and breached passwords | Users pick predictable passwords |
| **Do not force arbitrary periodic changes.** Change on evidence of compromise | Forced rotation leads to weak, predictable changes (`Summer2025!`) |
| **Allow all characters**, including spaces | Passphrases |
| **Limit guessing attempts** (lockout or throttling) | Stops online brute force |
| **Use MFA** for privileged and remote access | A password alone is not enough |
| **Prefer SSH keys** for servers | No password to guess |
| **Use a password manager** | Unique, random passwords per system |
| **Hash with a slow, salted algorithm** | Protects stored hashes |

> [!NOTE]
> Some compliance standards (PCI DSS, STIG, internal policy) still require periodic expiry or complexity. Follow the standard that applies to you, and document the reasons.

### How Linux stores passwords

Hashes live in `/etc/shadow` (readable only by root). Each hash begins with an **algorithm ID**:

| Prefix | Algorithm | Status |
|--------|-----------|--------|
| `$y$` | **yescrypt** | Modern default (recent Debian, Ubuntu, Fedora). Memory-hard |
| `$6$` | SHA-512 crypt | Good. Default on RHEL-based and SUSE-based for years |
| `$5$` | SHA-256 crypt | Acceptable |
| `$2b$` | bcrypt | Good (some systems) |
| `$1$` | MD5 | **Obsolete. Replace** |
| (no prefix, 13 chars) | DES crypt | **Obsolete** |

The hash includes a random **salt**, so identical passwords have different hashes. A stolen `shadow` file can still be attacked **offline**, so use long passwords and protect the file.

### Files and settings

| File | Purpose |
|------|---------|
| `/etc/login.defs` | Defaults for new accounts: aging, hash algorithm, UID ranges, `UMASK` |
| `/etc/security/pwquality.conf` | Password strength rules (used by `pam_pwquality`) |
| `/etc/security/faillock.conf` | Lockout rules |
| `/etc/pam.d/common-password` (Debian-based), `/etc/pam.d/system-auth`, `password-auth` (RHEL-based) | Where the modules are called |
| `/etc/shadow` | Per-user aging and the hash |

## Commands

### Aging with chage

```console
# chage -l alice
Last password change                                    : Mar 03, 2025
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password changes         : 0
Maximum number of days between password changes         : 99999
Number of days of warning before password expires       : 7

# chage -M 90 -m 1 -W 14 alice        # max 90 days, min 1 day, warn 14 days
# chage -I 30 alice                   # lock the account 30 days after the password expires
# chage -E 2025-12-31 alice           # account expiry (contractors)
# chage -d 0 alice                    # force a change at next login
# passwd -e alice                     # same: expire the password now
```

### Defaults for new users (/etc/login.defs)

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   14
ENCRYPT_METHOD  YESCRYPT          # or SHA512
UMASK           027
```

These apply only to **new** accounts. Existing accounts keep their values until you change them with `chage`.

### Password strength with pam_pwquality

`/etc/security/pwquality.conf`:

```
minlen = 14
minclass = 3
maxrepeat = 3
maxclassrepeat = 4
dictcheck = 1
usercheck = 1
enforcing = 1
enforce_for_root
retry = 3
```

| Setting | Meaning |
|---------|---------|
| `minlen` | Minimum length (credit rules reduce the effective count) |
| `minclass` | Minimum number of character classes (lower, upper, digit, other) |
| `dcredit`, `ucredit`, `lcredit`, `ocredit` | Credits/requirements per class. Negative values **require** that many (`dcredit = -1` = at least one digit) |
| `maxrepeat` | Maximum repeated identical characters |
| `dictcheck` | Reject dictionary words |
| `usercheck` | Reject passwords containing the user name |
| `enforce_for_root` | Apply to root as well |

Debian-based: enable the PAM module with `apt install libpam-pwquality` (it is added to `common-password`). Test a password: `pwscore` (reads from stdin), `pwmake 128` (generates a random password).

```console
$ echo 'Summer2025!' | pwscore
```

### Lockout (guessing protection)

Use `pam_faillock` as described in [PAM](02-pam.md). Also rate-limit and restrict SSH at the network level, and use `fail2ban`.

### Prevent password reuse

```
# in the password stack
password sufficient pam_unix.so use_authtok yescrypt shadow remember=5
```

Old hashes are stored in `/etc/security/opasswd`.

### Audit the accounts

```console
# awk -F: '($2 == "") {print $1}' /etc/shadow                 # empty passwords
# awk -F: '($2 ~ /^\$1\$/) {print $1}' /etc/shadow            # MD5 hashes (obsolete)
# awk -F: '($2 !~ /^[!*]/ && $2 !~ /^\$/) {print $1}' /etc/shadow    # odd hash formats
# for u in $(awk -F: '$3>=1000 {print $1}' /etc/passwd); do chage -l $u | grep -E 'Maximum|Account expires' | tr '\n' ' '; echo " $u"; done
# lastlog -b 90 | tail -n +2                                   # accounts that logged in more than 90 days ago (stale)
# passwd -S -a 2>/dev/null | awk '$2=="NP"'                    # accounts with no password
```

### Lock, unlock, disable

```console
# passwd -l alice            # lock the password (prefixes the hash with "!")
# passwd -u alice            # unlock
# usermod -e 1 alice         # expire the account (also blocks SSH key login via PAM account stage)
# usermod -s /usr/sbin/nologin alice
```

A **locked password does not block SSH key logins**. To disable a person completely, also expire the account or set a `nologin` shell, and remove their keys.

### Service and shared accounts

- Service accounts: no password (`passwd -l`), `nologin` shell, no SSH access.
- Automation: use keys or tokens with restricted rights, **not** passwords in scripts.
- Break-glass accounts: long random passwords in a sealed or vaulted location, with alerting on use.

### Check passwords against cracking (authorized testing only)

Run `john` or `hashcat` against your own hashes **only with authorization and under a controlled process** to find weak passwords. Handle the results as highly sensitive.

## Common pitfalls

- **Forcing 30-day changes with complexity rules.** Leads to weak patterns. Follow current guidance unless a standard requires otherwise.
- **Policy applied only to new accounts.** `login.defs` does not change existing users. Use `chage`.
- **Weak hashing algorithm left from old installations.** After changing `ENCRYPT_METHOD`, hashes update only when users change their passwords.
- **Empty passwords or `nullok`.**
- **Lockout without monitoring.** Attackers can lock out real users. Monitor lockouts and use throttling.
- **Locking a password but forgetting keys.** The user can still log in with SSH keys.
- **Passwords in command lines, scripts, or shell history.** Use prompts or secret stores.
- **Password reuse** between systems. Use unique passwords and a manager, or central identity.
- **No process for resets.** Verify identity before resetting, and force a change at the next login.
- **Ignoring the human side.** Training and easy-to-use tools (password managers, SSO) work better than stricter rules.

## Further reading

- [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [`pwquality.conf(5)`](https://man7.org/linux/man-pages/man5/pwquality.conf.5.html)
- [`chage(1)`](https://man7.org/linux/man-pages/man1/chage.1.html)
- [`login.defs(5)`](https://man7.org/linux/man-pages/man5/login.defs.5.html)
- [yescrypt](https://www.openwall.com/yescrypt/)
- [Have I Been Pwned: Pwned Passwords](https://haveibeenpwned.com/Passwords)
