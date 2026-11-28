# SSH Authentication and Key Management

## Overview

Creating a key pair is easy. **Managing keys over time** across many people and servers is the hard part. This page covers the lifecycle (create, distribute, rotate, revoke, audit) and the scalable alternatives: SSH certificates, central key lookup, and short-lived credentials. The basics of keys and `ssh-keygen` are in [SSH Keys and Authentication](../../part-3-networking/13-remote-access-firewalls-services/03-ssh-keys-and-authentication.md). Server settings are in [SSH Client and Server Configuration](../../part-3-networking/13-remote-access-firewalls-services/02-ssh-client-and-server-configuration.md).

## Key concepts

### The key lifecycle

| Stage | Good practice |
|-------|---------------|
| **Generate** | On the user's own device (never on the server). Ed25519 or a hardware-backed key (`ed25519-sk`). **Protect with a passphrase** |
| **Register** | A reviewed process: who owns this key, which accounts, which servers, expiry date |
| **Distribute** | Configuration management or a central source, not manual copying |
| **Use** | Agent with timeouts, `ProxyJump` through a bastion, no agent forwarding |
| **Rotate** | At fixed intervals and after a device is lost, replaced, or suspected compromised |
| **Revoke** | Immediately when someone leaves or a key is exposed |
| **Audit** | Regularly list who has access to what |

### Ways to authorize keys on servers

| Method | How | Scales? |
|--------|-----|---------|
| **`authorized_keys` files** | Copy public keys into `~/.ssh/authorized_keys` | Poorly. Keys accumulate and are forgotten |
| **Configuration-managed keys** | Ansible/Puppet manage `authorized_keys` (with `exclusive` mode that removes unknown keys) | Good |
| **`AuthorizedKeysCommand`** | sshd runs a program that returns the user's keys (from LDAP, FreeIPA, SSSD, an API) | Very good. Central revoke |
| **SSH certificates (CA)** | Servers trust a CA. The CA signs user keys for a limited time and set of principals | Best for larger fleets. Short-lived, no per-server key lists |
| **Kerberos / GSSAPI** | Single sign-on in domains | Good in AD/FreeIPA environments |
| **Access platforms** (Teleport, Boundary, Smallstep, cloud OS Login / SSM Session Manager) | Short-lived credentials, MFA, session recording | Strong audit |

## Commands and examples

### 1. Inventory and audit keys

```console
# for f in /home/*/.ssh/authorized_keys /root/.ssh/authorized_keys; do [ -f "$f" ] && echo "== $f" && ssh-keygen -lf "$f"; done
== /home/alice/.ssh/authorized_keys
256 SHA256:Qh0Jc2m1b3ZyJ7n0Gk4r2U9d0v5xYpC1sKzT8aLw3Ho alice@laptop (ED25519)
3072 SHA256:7f3m...  old-key-2019 (RSA)

# sshd -T | grep -E 'authorizedkeysfile|authorizedkeyscommand|trustedusercakeys|revokedkeys|authorizedprincipals'
# journalctl -u sshd | grep 'Accepted publickey' | awk '{print $9, $11, $16}' | sort | uniq -c    # fingerprints actually used (needs LogLevel VERBOSE)
```

Review: unknown keys, RSA keys under 3072 bits, DSA keys, keys without comments, keys for departed staff, keys in service accounts, and the **same key on many accounts**. Remove what is not needed.

`LogLevel VERBOSE` in `sshd_config` records the **fingerprint** of each key used, so you can tie a login to a specific key.

### 2. Restrict what a key can do

```
# ~/.ssh/authorized_keys on the server
restrict,from="192.0.2.0/24",command="/usr/local/bin/backup-only" ssh-ed25519 AAAA... backup@jobs
restrict,port-forwarding,permitopen="db.internal:5432" ssh-ed25519 AAAA... tunnel@appserver
expiry-time="20251231" ssh-ed25519 AAAA... contractor@example.com
```

### 3. Central keys with AuthorizedKeysCommand

```
# /etc/ssh/sshd_config.d/20-keys-command.conf
AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
AuthorizedKeysCommandUser nobody
```

SSSD fetches the public keys stored in LDAP, FreeIPA, or AD (`sshPublicKey` attribute). The command and its directories must be owned by root and not writable by others, and the command user must be an unprivileged account. Add a custom script for other sources (a web API, a database). Make sure the script **fails closed** and is fast.

```console
# sshd -t && systemctl reload sshd
$ /usr/bin/sss_ssh_authorizedkeys alice
```

### 4. SSH certificates

A **certificate** is a public key signed by a **CA key**. Servers trust the CA. They then accept any valid certificate signed by it, with no per-user key files.

**Create the CA (keep it offline or in an HSM/vault):**

```console
$ ssh-keygen -t ed25519 -f ssh_user_ca -C "Example user CA"
```

**Sign a user's public key** (identity string, allowed login names (principals), validity):

```console
$ ssh-keygen -s ssh_user_ca -I alice-laptop-2025-03 -n alice,deploy -V +8h /home/alice/.ssh/id_ed25519.pub
Signed user key /home/alice/.ssh/id_ed25519-cert.pub: id "alice-laptop-2025-03" serial 0 for alice,deploy valid from ... to ...
$ ssh-keygen -Lf ~/.ssh/id_ed25519-cert.pub        # inspect it
```

**Trust the CA on servers:**

```
# /etc/ssh/sshd_config.d/30-ca.conf
TrustedUserCAKeys /etc/ssh/ssh_user_ca.pub
AuthorizedPrincipalsFile /etc/ssh/principals/%u        # which principals may log in as this account
RevokedKeys /etc/ssh/revoked_keys                       # optional revocation list
```

```console
# mkdir -p /etc/ssh/principals && echo alice > /etc/ssh/principals/alice && echo deploy >> /etc/ssh/principals/deploy_user
```

| Benefit | Detail |
|---------|--------|
| **Short-lived** access | A certificate valid for 8 hours needs no revocation |
| **No per-server key distribution** | New servers only need the CA public key |
| **Principals** | Map people/roles to accounts centrally |
| **Auditable** | The certificate `-I` identity appears in the logs |
| **Host certificates** | The CA also signs **host keys**, so clients verify servers without "trust on first use" (`HostCertificate`, and `@cert-authority` in `known_hosts`) |

Tools that automate issuance after SSO and MFA: HashiCorp Vault / OpenBao SSH secrets engine, Smallstep `step-ca`, Teleport, cloud-provider tools.

### 5. Revoke keys and certificates

```console
# ssh-keygen -k -f /etc/ssh/revoked_keys old-key.pub          # create a revocation list (KRL); add with -u to update
# ssh-keygen -Qf /etc/ssh/revoked_keys suspect-key.pub        # check whether a key is revoked
```

Revocation checklist when a person leaves or a key is lost:

1. Disable the identity in the central source (AD/LDAP/FreeIPA/IdP).
2. Remove their keys from `authorized_keys`, central stores, and deploy keys (Git hosts, CI).
3. Revoke certificates (KRL) if they have long validity.
4. Rotate shared secrets they knew.
5. Review logs for their last activity.
6. Remove the account (or lock and expire it) and its `sudo` rights.

### 6. Bastion (jump host) pattern

Allow SSH to production servers **only from a hardened bastion**. Everyone connects through it with `ProxyJump`, and sessions are logged there.

```
# ~/.ssh/config
Host prod-*
    ProxyJump bastion.example.com
    User alice
```

On servers, firewall port 22 to the bastion's address. Add MFA and session recording on the bastion.

### 7. Agent hygiene

```console
$ ssh-add -t 1h ~/.ssh/id_ed25519            # key expires in the agent after 1 hour
$ ssh-add -l                                  # what is loaded
$ ssh-add -D                                  # clear all
```

Avoid `ForwardAgent yes`. Use `ProxyJump`. If you must forward an agent, use a **confirmation** key: `ssh-add -c KEY`.

### 8. Hardware-backed keys and Kerberos

```console
$ ssh-keygen -t ed25519-sk -O resident -O verify-required -C "alice yubikey"      # FIDO2 key; needs a touch (and PIN)
$ ssh-keygen -K                                                                    # download resident keys from the token
```

For Kerberos environments (AD, FreeIPA), enable GSSAPI for passwordless SSO with a valid ticket:

```
# sshd_config
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
# client ~/.ssh/config
GSSAPIAuthentication yes
GSSAPIDelegateCredentials no
```

```console
$ kinit alice@EXAMPLE.COM && ssh server01.example.com
```

### 9. Require more than one method

```
# sshd_config: key AND second factor
AuthenticationMethods publickey,keyboard-interactive
```

See [Multi-Factor Authentication Concepts](07-multi-factor-authentication-concepts.md).

### Policy checklist

| Control | Standard |
|---------|----------|
| Key type | Ed25519 (or RSA 3072+), no DSA, no passphrase-less keys on laptops |
| Authorization | Central source or CA, not scattered `authorized_keys` |
| Lifetime | Certificates: hours to days. Plain keys: 1 year or less, with an owner |
| Root login | Disabled. Use `sudo` |
| Service keys | Separate, restricted (`command=`, `from=`), stored in a secrets manager |
| Logging | `LogLevel VERBOSE`, central log storage, alert on new keys |
| Offboarding | Written procedure, tested |

## Common pitfalls

- **Keys that live forever.** Employees leave, but their keys remain. Use expiry and audits.
- **Private keys copied between machines or stored on servers.** Each device gets its own key.
- **Using the same key for everything** (personal, work, automation). One leak is a disaster.
- **Passphrase-less private keys** left on shared systems.
- **Agent forwarding to untrusted hosts.** Anyone with root there can use your agent.
- **`AuthorizedKeysCommand` that returns keys when the backend is down** or is run as a privileged user. Run it as `nobody`, keep scripts root-owned.
- **No logging of key fingerprints.** You cannot tell whose key was used.
- **Certificates without a short lifetime** are just keys with extra steps. Keep them short.
- **Protecting the CA key poorly.** Anyone with the CA key can access everything that trusts it. Store it offline or in an HSM/vault.
- **Forgetting deploy keys and CI keys** in the leaver process.
- **Weak legacy keys.** RSA 1024/2048-bit and DSA keys should be replaced.

## Further reading

- [`ssh-keygen(1)`: certificates](https://man7.org/linux/man-pages/man1/ssh-keygen.1.html)
- [`sshd_config(5)`](https://man7.org/linux/man-pages/man5/sshd_config.5.html)
- [OpenSSH certificates (cookbook)](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Certificate-based_Authentication)
- [Smallstep: SSH certificates](https://smallstep.com/docs/ssh/)
- [HashiCorp Vault / OpenBao SSH secrets engine](https://openbao.org/docs/secrets/ssh/)
- [NIST IR 7966: SSH key management](https://csrc.nist.gov/publications/detail/nistir/7966/final)
