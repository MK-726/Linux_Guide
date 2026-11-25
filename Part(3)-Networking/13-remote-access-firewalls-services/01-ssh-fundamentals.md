# SSH Fundamentals

## Overview

**SSH** (Secure Shell) is the standard way to administer Linux servers remotely. It provides an **encrypted**, **authenticated** channel for a remote shell, file transfer, and port forwarding. **OpenSSH** is the implementation used on nearly all Linux systems. Replace `telnet`, `rsh`, and `ftp` with SSH, because those send passwords and data in clear text.

## Key concepts

### Client and server

| Component | Program | Service | Port |
|-----------|---------|---------|------|
| **Client** | `ssh`, `scp`, `sftp`, `ssh-keygen`, `ssh-agent` | — | Connects to TCP **22** |
| **Server** | `sshd` | `sshd.service` (RHEL-based, SUSE-based), `ssh.service` (Debian-based) | Listens on TCP **22** |

Install the server: **Debian-based** `apt install openssh-server`, **RHEL-based** `dnf install openssh-server`. Most cloud images have it already.

### What SSH protects

| Protection | How |
|------------|-----|
| **Confidentiality** | Traffic is encrypted (modern ciphers such as ChaCha20-Poly1305 and AES-GCM) |
| **Integrity** | Messages are authenticated; tampering is detected |
| **Server authentication** | The server proves its identity with a **host key** |
| **User authentication** | Password, public key, or other methods |

### How a connection starts

```
1. Client connects to port 22; both sides agree on protocol version 2 and algorithms.
2. Key exchange creates a shared session key (forward secrecy).
3. Server proves its identity with its HOST KEY (the client checks known_hosts).
4. Client authenticates: public key, password, etc.
5. Encrypted session: shell, command, file transfer, or forwarding.
```

### Host keys and known_hosts

The first time you connect to a server, SSH shows its **fingerprint** and asks you to confirm:

```console
$ ssh alice@server01.example.com
The authenticity of host 'server01.example.com (192.0.2.10)' can't be established.
ED25519 key fingerprint is SHA256:Qh0Jc2m1b3ZyJ7n0Gk4r2U9d0v5xYpC1sKzT8aLw3Ho.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Compare the fingerprint with one you got **through a trusted channel** (cloud console, the administrator, the server's console). If you answer `yes`, the key is saved in `~/.ssh/known_hosts` and checked on every later connection.

Get the fingerprint on the server:

```console
# ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:Qh0Jc2m1b3ZyJ7n0Gk4r2U9d0v5xYpC1sKzT8aLw3Ho root@server01 (ED25519)
```

If the key **changes**, SSH refuses to connect:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

| Possible cause | Action |
|----------------|--------|
| The server was **reinstalled** or rebuilt (new host keys) | Verify with the administrator, then remove the old key: `ssh-keygen -R server01.example.com` |
| The IP now belongs to a different server (cloud, DHCP) | Same |
| A **man-in-the-middle attack** | Do **not** connect. Investigate |

Never disable host key checking permanently (`StrictHostKeyChecking no`) on a production system: you lose protection against impersonation.

## Commands

### Connect

```console
$ ssh server01                          # same user name as locally
$ ssh alice@server01.example.com
$ ssh -p 2222 alice@192.0.2.10          # another port
$ ssh -i ~/.ssh/id_ed25519 alice@server01        # a specific key
$ ssh alice@server01 'uptime'           # run one command and exit
$ ssh alice@server01 'sudo systemctl status nginx'
$ ssh -t alice@server01 'sudo -i'       # -t: allocate a terminal (needed for interactive commands)
$ ssh -v alice@server01                 # verbose (-vv, -vvv: more): the first debugging tool
$ ssh -4 / -6 alice@server01            # force IPv4 / IPv6
$ ssh -J jump.example.com alice@10.0.0.5     # via a jump host
```

| Option | Purpose |
|--------|---------|
| `-p PORT` | Server port |
| `-i FILE` | Identity (private key) file |
| `-l USER` | User name (same as `user@host`) |
| `-v` | Debug output |
| `-t` / `-T` | Force / disable terminal allocation |
| `-N` | No remote command (for tunnels) |
| `-f` | Go to the background after authentication |
| `-o 'Option=value'` | Any configuration option on the command line |
| `-q` | Quiet |
| `-J HOST` | ProxyJump |
| `-F FILE` | Use another client configuration file |

### Escape sequences (when the session hangs)

Press **Enter**, then:

| Keys | Action |
|------|--------|
| `~.` | Disconnect (kill a frozen session) |
| `~?` | Help |
| `~C` | Command line (add port forwards) |
| `~~` | Send a literal `~` |

### File transfer

| Tool | Notes |
|------|-------|
| `scp` | `scp file.txt alice@server01:/tmp/`. Simple copy. Modern OpenSSH (9.0+) uses the SFTP protocol internally |
| `sftp` | Interactive FTP-like client over SSH: `sftp alice@server01` (`ls`, `get`, `put`, `cd`, `lcd`) |
| `rsync -e ssh` | **Best for syncing** directories: only changes are sent |

```console
$ scp report.pdf alice@server01:/home/alice/
$ scp -r project/ alice@server01:/srv/
$ scp alice@server01:/var/log/app.log .
$ rsync -avz --progress -e 'ssh -p 2222' project/ alice@server01:/srv/project/
$ rsync -avz --delete --dry-run project/ alice@server01:/srv/project/     # preview first
$ sftp alice@server01
sftp> put backup.tar.gz
sftp> get /etc/hostname
```

> [!WARNING]
> `rsync --delete` removes files at the destination that do not exist at the source. A wrong source path or a missing trailing slash can delete data. Use `--dry-run` first.

### Server basics

```console
# systemctl enable --now sshd            # RHEL-based; Debian-based: ssh
# systemctl status sshd
# ss -tlnp | grep sshd
# journalctl -u sshd -f                  # live log (Debian-based: -u ssh)
# grep sshd /var/log/auth.log            # Debian-based; RHEL-based: /var/log/secure
$ who                                    # who is logged in
$ last -i | head                         # recent logins
```

### Host keys on the server

```console
# ls -l /etc/ssh/ssh_host_*
-rw------- 1 root root  399 ... ssh_host_ed25519_key
-rw-r--r-- 1 root root   93 ... ssh_host_ed25519_key.pub
```

Cloned virtual machines share host keys unless you regenerate them. Regenerate with `rm /etc/ssh/ssh_host_* && ssh-keygen -A` (Debian-based: `dpkg-reconfigure openssh-server`) and restart `sshd`. Cloud-init often does this on first boot.

### Look up and manage known hosts

```console
$ ssh-keygen -F server01.example.com        # find in known_hosts
$ ssh-keygen -R server01.example.com        # remove old entries
$ ssh-keyscan -t ed25519 server01.example.com    # fetch a server's host key (verify the fingerprint yourself!)
```

## Common pitfalls

- **Accepting the fingerprint blindly.** The first connection is the moment of trust. Verify it when security matters.
- **`REMOTE HOST IDENTIFICATION HAS CHANGED`** after a rebuild. Remove the old key, but only after checking the cause.
- **"Permission denied (publickey)".** The server accepts only keys and yours is not authorized (see [SSH Keys and Authentication](03-ssh-keys-and-authentication.md)).
- **"Connection refused".** `sshd` not running, wrong port, or a firewall reject rule. **"Connection timed out"**: firewall drop or routing.
- **Frozen sessions.** Use `~.`, and set `ServerAliveInterval` (see [SSH configuration](02-ssh-client-and-server-configuration.md)).
- **Missing `-t`** for interactive commands over `ssh host command`: `sudo` or `top` complain about no terminal.
- **`scp` quirks** (remote shell expansion, wildcards, directories). Prefer `rsync` or `sftp` for complex copies.
- **Logging in as root directly.** Use a normal user and `sudo`.
- **Using SSH with passwords for all users on internet-facing servers.** Brute-force attacks start within minutes. Use keys.

## Further reading

- [`ssh(1)`](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [`sshd(8)`](https://man7.org/linux/man-pages/man8/sshd.8.html)
- [OpenSSH](https://www.openssh.com)
- [`rsync(1)`](https://man7.org/linux/man-pages/man1/rsync.1.html)
