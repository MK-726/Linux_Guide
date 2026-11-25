# SSH Keys and Authentication

## Overview

**Public-key authentication** is more secure and more convenient than passwords. You create a **key pair**: a **private key** (secret, stays on your machine) and a **public key** (placed on servers). The server challenges you to prove you hold the private key, without ever sending it.

## Key concepts

### Key types

| Type | Command option | Notes |
|------|----------------|-------|
| **Ed25519** | `-t ed25519` | **Recommended.** Small, fast, strong |
| RSA | `-t rsa -b 4096` | Widely compatible. Use at least 3072 bits (4096 preferred) |
| ECDSA | `-t ecdsa -b 384` | Supported, less commonly chosen |
| **FIDO2 / security key** | `-t ed25519-sk` | Private key lives on a hardware token (YubiKey). Needs a touch to use |
| DSA | — | Obsolete. Do not use |

The old `ssh-rsa` **signature algorithm (SHA-1)** is disabled by default in current OpenSSH. RSA keys still work with `rsa-sha2-256/512`. If a very old server rejects modern keys, upgrade the server rather than re-enabling weak algorithms.

### Files

| File | Content | Permissions |
|------|---------|-------------|
| `~/.ssh/id_ed25519` | **Private key** | `600` (SSH refuses to use it otherwise) |
| `~/.ssh/id_ed25519.pub` | Public key | `644` |
| `~/.ssh/authorized_keys` (server) | Public keys allowed to log in as this user | `600` (or `644`), owned by the user |
| `~/.ssh/known_hosts` | Server host keys you trust | |
| `~/.ssh/config` | Client configuration | `600` |
| `~/.ssh` directory | | `700` |

> [!IMPORTANT]
> **Never share or copy the private key** to servers. Never email it or commit it to Git. Protect it with a **passphrase**.

## Commands

### Create a key pair

```console
$ ssh-keygen -t ed25519 -C "alice@laptop 2025"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/alice/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Your identification has been saved in /home/alice/.ssh/id_ed25519
Your public key has been saved in /home/alice/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:abc123... alice@laptop 2025
```

| Option | Meaning |
|--------|---------|
| `-t TYPE` | Key type |
| `-b BITS` | Key size (RSA) |
| `-C COMMENT` | Label to identify the key |
| `-f FILE` | Output file (use different names for different purposes) |
| `-N ''` | Empty passphrase (only for automation with other protections) |
| `-a 100` | KDF rounds for passphrase protection (default is already good in new versions) |

Other `ssh-keygen` uses:

```console
$ ssh-keygen -lf ~/.ssh/id_ed25519.pub         # show the fingerprint
$ ssh-keygen -p -f ~/.ssh/id_ed25519           # change the passphrase
$ ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub     # re-create the public key
$ ssh-keygen -t ed25519-sk -O resident         # hardware-backed key
```

### Install the public key on a server

```console
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@server01
$ ssh alice@server01                            # now logs in with the key
```

`ssh-copy-id` appends the public key to `~/.ssh/authorized_keys` on the server and sets safe permissions. Manual method:

```console
$ cat ~/.ssh/id_ed25519.pub | ssh alice@server01 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

### The authorized_keys file

One public key per line, optionally with **restrictions** in front:

```
ssh-ed25519 AAAAC3Nza...  alice@laptop 2025
from="192.0.2.0/24",no-port-forwarding,no-agent-forwarding,no-X11-forwarding ssh-ed25519 AAAAC3Nza... backup-key
command="/usr/local/bin/backup-only.sh",restrict ssh-ed25519 AAAAC3Nza... backup-automation
```

| Option | Effect |
|--------|--------|
| `from="PATTERN"` | Key works only from these addresses |
| `command="CMD"` | **Only** this command runs with this key (good for automation) |
| `restrict` | Disable all extra features (forwarding, PTY, X11). Add back only what is needed |
| `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty` | Individual restrictions |
| `expiry-time="20251231"` | Key expires |

### ssh-agent: type the passphrase once

The agent holds decrypted keys in memory for the session.

```console
$ eval "$(ssh-agent -s)"
Agent pid 4521
$ ssh-add ~/.ssh/id_ed25519
Enter passphrase for /home/alice/.ssh/id_ed25519:
Identity added: /home/alice/.ssh/id_ed25519
$ ssh-add -l                      # list loaded keys (fingerprints)
$ ssh-add -D                      # remove all keys
$ ssh-add -t 3600 ~/.ssh/id_ed25519       # key expires from the agent after 1 hour
```

Desktop sessions and many distributions start an agent automatically. `AddKeysToAgent yes` in `~/.ssh/config` loads keys on first use.

### Agent forwarding: be careful

`ssh -A` (or `ForwardAgent yes`) lets the remote host use your agent to authenticate onward. **Anyone with root on that host can use your keys while you are connected.** Prefer **`ProxyJump`**, which never exposes the agent:

```console
$ ssh -J bastion.example.com alice@10.0.0.5
```

### SSH certificates (for larger environments)

Instead of copying public keys everywhere, a **certificate authority (CA)** signs keys. Servers trust the CA, so adding or removing access needs no change on servers.

```console
$ ssh-keygen -t ed25519 -f ssh_ca                                  # create the CA key (guard it!)
$ ssh-keygen -s ssh_ca -I alice-2025 -n alice,deploy -V +12w ~/.ssh/id_ed25519.pub     # sign alice's key for 12 weeks
# On servers (sshd_config):
TrustedUserCAKeys /etc/ssh/ssh_ca.pub
```

Certificates also expire automatically. A matching host-certificate setup removes the "trust on first use" question for host keys.

### Two-factor authentication

| Method | Notes |
|--------|-------|
| **FIDO2 security key** (`ed25519-sk`) | Phishing-resistant. Touch required |
| **PAM TOTP** (Google Authenticator / `libpam-google-authenticator`) | Combine with `AuthenticationMethods publickey,keyboard-interactive` |
| **Certificates from an identity provider** | Short-lived, tied to single sign-on |

### Troubleshooting key authentication

```console
$ ssh -vvv alice@server01 2>&1 | grep -E 'Offering|Server accepts|Authentication|denied|identity'
debug1: Offering public key: /home/alice/.ssh/id_ed25519 ED25519 SHA256:abc...
debug1: Authentications that can continue: publickey
alice@server01: Permission denied (publickey).
```

Checklist (on the **server**, as root):

| Check | Command |
|-------|---------|
| Server log (the real reason) | `journalctl -u sshd -e` or `/var/log/auth.log`, `/var/log/secure` |
| Permissions | `namei -l /home/alice/.ssh/authorized_keys`; the home directory must **not be group- or world-writable**; `.ssh` `700`; `authorized_keys` `600` or `644`; owned by the user |
| Key is in the file | `ssh-keygen -lf ~/.ssh/authorized_keys` lists the fingerprint of each key; compare with `ssh-keygen -lf ~/.ssh/id_ed25519.pub` on the client |
| Key type allowed | `sshd -T \| grep -i pubkeyacceptedalgorithms` |
| Account restrictions | `AllowUsers`, `AllowGroups`, `Match` blocks, locked or expired account (`passwd -S alice`, `chage -l alice`) |
| SELinux context | `restorecon -Rv /home/alice/.ssh` |
| Shell valid | `getent passwd alice` (the shell must be listed in `/etc/shells` or a real shell) |
| Client offers the key | `ssh -i key -o IdentitiesOnly=yes -vvv` |

### Key hygiene

- Use a **separate key per device** and per purpose (work, personal, automation).
- Protect keys with **passphrases**, and use the agent.
- **Rotate** keys periodically and **remove** keys of people who leave: audit `authorized_keys` on servers.
- Keep **private keys off shared servers**. Use agent forwarding alternatives or certificates.
- Use `from=` and `command=` restrictions for automation keys.
- Back up private keys securely (encrypted). Losing a key without a second path means lockout.

## Common pitfalls

- **Wrong permissions** (`.ssh`, `authorized_keys`, or the home directory). `StrictModes` makes the server ignore the key silently. Check the server log.
- **Copying the private key to the server** instead of the public key (`.pub`).
- **Key without a passphrase** stored on laptops and servers. If stolen, it works immediately.
- **Line breaks in `authorized_keys`.** One key must be on **one line**.
- **Too many keys in the agent** leads to "Too many authentication failures". Use `IdentitiesOnly yes` and `IdentityFile`.
- **Locking out by disabling passwords first.** Test key login in a second session, then disable passwords.
- **Using old RSA keys with SHA-1 on new servers.** Upgrade keys to Ed25519 or ensure the key is accepted with `rsa-sha2-*`.
- **Home directory on NFS or encrypted home** not mounted at login. The key file is not readable at the time of authentication.
- **Shared accounts with shared keys.** You lose accountability. Use individual accounts and `sudo`.
- **Leaving keys of former staff** in `authorized_keys`.

## Further reading

- [`ssh-keygen(1)`](https://man7.org/linux/man-pages/man1/ssh-keygen.1.html)
- [`ssh-agent(1)`](https://man7.org/linux/man-pages/man1/ssh-agent.1.html)
- [`sshd(8)`: AUTHORIZED_KEYS FILE FORMAT](https://man7.org/linux/man-pages/man8/sshd.8.html)
- [OpenSSH certificates](https://man7.org/linux/man-pages/man1/ssh-keygen.1.html#CERTIFICATES)
