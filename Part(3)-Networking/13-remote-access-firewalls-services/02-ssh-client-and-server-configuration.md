# SSH Client and Server Configuration

## Overview

SSH behavior is controlled by two configuration systems: the **client** (`ssh_config`, per user in `~/.ssh/config`) and the **server** (`sshd_config`). A good client configuration saves typing and avoids mistakes. A good server configuration makes SSH much harder to attack.

## Key concepts

### Client configuration files

| File | Scope |
|------|-------|
| `~/.ssh/config` | Per user. Permissions `600` |
| `/etc/ssh/ssh_config` and `/etc/ssh/ssh_config.d/*.conf` | System-wide defaults |
| `~/.ssh/known_hosts` | Known server host keys |
| `~/.ssh/authorized_keys` | (On the **server**) public keys allowed to log in |

The client uses the **first value found** for each option (command line first, then `~/.ssh/config`, then system files). Settings in `Host` blocks apply when the host name you type matches.

### Server configuration files

| File | Purpose |
|------|---------|
| `/etc/ssh/sshd_config` | Main server configuration |
| `/etc/ssh/sshd_config.d/*.conf` | Drop-in files (included at the **top** of the main file on most distributions) |
| `/etc/ssh/ssh_host_*_key` | Host private keys |

> [!IMPORTANT]
> For `sshd`, the **first** occurrence of an option wins. Drop-in files in `sshd_config.d/` are read before the rest of the main file, so a drop-in overrides the main file. Check the effective settings with `sshd -T`.

## Commands and examples

### Client: ~/.ssh/config

```
# ~/.ssh/config
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 3
    AddKeysToAgent yes
    IdentitiesOnly yes

Host web1
    HostName 192.0.2.10
    User alice
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host db1
    HostName 10.0.0.5
    User dbadmin
    ProxyJump bastion
    IdentityFile ~/.ssh/id_ed25519_work

Host bastion
    HostName bastion.example.com
    User alice
    ForwardAgent no

Host *.internal.example.com
    User alice
    ProxyJump bastion
```

Now `ssh web1` and `ssh db1` work. Because `db1` has `ProxyJump bastion`, the connection goes through the bastion host automatically.

| Option | Meaning |
|--------|---------|
| `Host` | Pattern (`*`, `?`, comma list, `!` to negate) |
| `HostName` | Real host name or IP |
| `User` | Login name |
| `Port` | Server port |
| `IdentityFile` | Private key to offer |
| `IdentitiesOnly yes` | Offer only the listed keys (avoids "Too many authentication failures") |
| `ProxyJump` | Jump host(s) (comma separated) |
| `ServerAliveInterval` / `ServerAliveCountMax` | Keep-alive messages and give-up count |
| `ForwardAgent` | Forward the ssh-agent (use rarely, see warnings) |
| `LocalForward`, `RemoteForward`, `DynamicForward` | Tunnels (see [Tunneling](04-ssh-tunneling-and-port-forwarding.md)) |
| `StrictHostKeyChecking` | `ask` (default), `yes`, `accept-new`, `no` |
| `UserKnownHostsFile` | Alternate known-hosts file |
| `Compression yes` | Compress (useful on slow links) |
| `ControlMaster auto`, `ControlPath`, `ControlPersist 10m` | **Connection sharing**: later sessions reuse one connection (much faster) |
| `LogLevel` | `QUIET`, `INFO`, `VERBOSE`, `DEBUG` |

Connection multiplexing:

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

Check or stop a shared connection: `ssh -O check web1`, `ssh -O exit web1`.

Show the **effective** configuration for a host:

```console
$ ssh -G web1 | grep -i -E '^(hostname|user|port|identityfile|proxyjump)'
hostname 192.0.2.10
user alice
port 22
```

### Server: /etc/ssh/sshd_config

```
# /etc/ssh/sshd_config.d/10-hardening.conf
Port 22
AddressFamily any
ListenAddress 0.0.0.0

PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
UsePAM yes

AllowGroups sshusers
MaxAuthTries 3
LoginGraceTime 30
MaxSessions 10
ClientAliveInterval 300
ClientAliveCountMax 2

X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no

LogLevel VERBOSE
Banner /etc/issue.net
```

| Option | Meaning |
|--------|---------|
| `Port`, `ListenAddress` | Where to listen |
| `PermitRootLogin` | `no`, `prohibit-password` (keys only, the usual default), `forced-commands-only`, `yes` |
| `PubkeyAuthentication` | Allow key login |
| `PasswordAuthentication` | Allow password login. Set `no` once keys work |
| `KbdInteractiveAuthentication` | Interactive prompts (PAM, 2FA). Passwords can also come through here, so disable it too if you disable passwords |
| `AllowUsers` / `AllowGroups` | Only these users or groups may log in |
| `DenyUsers` / `DenyGroups` | Block these |
| `MaxAuthTries` | Authentication attempts per connection |
| `LoginGraceTime` | Seconds to log in |
| `ClientAliveInterval` / `ClientAliveCountMax` | Server keep-alive: drop dead clients |
| `X11Forwarding`, `AllowAgentForwarding`, `AllowTcpForwarding` | Features that expand attack surface. Disable if not needed |
| `Banner` | Message before login |
| `Subsystem sftp internal-sftp` | SFTP service |
| `StrictModes yes` | Check permissions of the user's key files |
| `LogLevel VERBOSE` | Logs key fingerprints used (useful for audits) |

### Match blocks: different rules for different users

```
Match Group sftponly
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no

Match Address 192.0.2.0/24
    PasswordAuthentication yes
```

`Match` applies to the following lines until the next `Match` or the end of the file. Put `Match` blocks **last**.

### Test and apply safely (do this every time)

```console
# sshd -t                                      # check syntax (silent when OK)
# sshd -T | grep -i -E 'permitrootlogin|passwordauthentication|allowgroups'     # effective values
# systemctl reload sshd                        # apply; existing sessions stay open (Debian-based: ssh)
$ ssh -v alice@server01                        # test from a SECOND terminal before closing the first
```

**Never close your working session until a new login succeeds.**

### Changing the port

```
Port 2222
```

Also:

- Allow the new port in the firewall.
- **SELinux** (RHEL-based): `semanage port -a -t ssh_port_t -p tcp 2222`.
- Update client configs (`Port 2222`).
- Do **not** remove the firewall rule for 22 until the new port works.

> [!NOTE]
> On Ubuntu releases that start SSH through **socket activation** (`ssh.socket`), `Port` and `ListenAddress` in `sshd_config` may be ignored. Edit the socket (`systemctl edit ssh.socket` and set `ListenStream=`), or disable socket activation (`systemctl disable --now ssh.socket && systemctl enable --now ssh.service`). Check with `systemctl status ssh.socket`.

A non-standard port reduces log noise from bots but is **not a security measure**. Keys, patching, and a firewall are.

### Hardening checklist

| Measure | How |
|---------|-----|
| Key-only login | `PasswordAuthentication no` after verifying a key login |
| No direct root login | `PermitRootLogin no` |
| Limit users | `AllowGroups sshusers` |
| Limit source addresses | Firewall rules, or `Match Address`, or a VPN/bastion |
| Strong algorithms | Keep OpenSSH updated. Use the defaults (modern OpenSSH removes weak ones) |
| Rate limiting / banning | `fail2ban`, `ufw limit ssh`, firewall rate limits |
| Second factor | PAM with TOTP, hardware keys (FIDO2 `sk` keys), or SSH certificates |
| Disable unused features | X11, agent, TCP forwarding |
| Audit | `LogLevel VERBOSE`, central logs |
| Keep software current | Update `openssh-server` promptly |

### Logs

```console
# journalctl -u sshd --since today | grep -i -E 'failed|invalid|accepted'
# grep 'Accepted publickey' /var/log/auth.log           # (RHEL-based: /var/log/secure)
```

## Common pitfalls

- **Locking yourself out.** Test with a **second session**, and have console access.
- **Drop-in files silently overriding `sshd_config`** (first value wins). Use `sshd -T` to see effective values.
- **Disabling `PasswordAuthentication` without working keys** for every user who needs access.
- **Disabling only `PasswordAuthentication`** but leaving `KbdInteractiveAuthentication yes` with PAM. Passwords may still work. Check with `sshd -T`.
- **`AllowUsers` or `AllowGroups` too strict**, locking out service or emergency accounts.
- **Permissions too open on `~/.ssh` or the home directory.** `StrictModes` rejects keys (see the next page).
- **Using `reload` vs. `restart` incorrectly.** `reload` re-reads the configuration and keeps sessions. A full `restart` is rarely needed.
- **`ForwardAgent yes` for all hosts** lets root on a compromised server use your keys. Use `ProxyJump` instead.
- **SELinux blocking a new port** or the home-directory key files. Use `restorecon -Rv ~/.ssh` and `semanage port`.
- **Cloud security group not updated** for the new port.
- **Trailing configuration in `Match` blocks** that applies to all later settings. Keep `Match` at the end.

## Further reading

- [`ssh_config(5)`](https://man7.org/linux/man-pages/man5/ssh_config.5.html)
- [`sshd_config(5)`](https://man7.org/linux/man-pages/man5/sshd_config.5.html)
- [Mozilla OpenSSH guidelines](https://infosec.mozilla.org/guidelines/openssh)
- [fail2ban](https://github.com/fail2ban/fail2ban)
