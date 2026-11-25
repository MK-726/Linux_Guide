# SSH Tunneling and Port Forwarding

## Overview

SSH can carry **other network traffic** through its encrypted connection. This is called **port forwarding** or **tunneling**. It lets you reach a service that is not exposed to the network, protect an unencrypted protocol, or use a jump host. It is powerful, which also means it can bypass firewalls and policies. Use it deliberately.

## Key concepts

Three forwarding types:

| Type | Option | Direction | Typical use |
|------|--------|-----------|-------------|
| **Local** | `-L` | A port on **your** machine → a destination reachable from the SSH server | Reach an internal database or web UI |
| **Remote** | `-R` | A port on the **SSH server** → a destination reachable from your machine | Expose a local service temporarily, reverse access behind NAT |
| **Dynamic** | `-D` | A local **SOCKS proxy** → any destination via the SSH server | Browse or run tools "as if" from the server's network |

### Local forwarding (-L)

`ssh -L [bind_address:]LOCALPORT:DESTHOST:DESTPORT user@sshserver`

```
Your laptop                      SSH server (bastion)           Database server
localhost:5433  ──[ encrypted ]──►  bastion  ──────────────────►  db.internal:5432
```

```console
$ ssh -N -L 5433:db.internal:5432 alice@bastion.example.com
# In another terminal:
$ psql -h 127.0.0.1 -p 5433 -U app mydb
```

`DESTHOST` is resolved **from the SSH server's point of view**. Use `localhost` if the service runs on the SSH server itself:

```console
$ ssh -N -L 8080:localhost:80 alice@server01        # your 8080 → server01's own port 80
```

### Remote forwarding (-R)

`ssh -R [bind_address:]REMOTEPORT:DESTHOST:DESTPORT user@sshserver`

```
Your laptop                       SSH server
localhost:3000  ◄──[ encrypted ]──  server01:9000
```

```console
$ ssh -N -R 9000:localhost:3000 alice@server01.example.com
# On server01: curl http://localhost:9000  → reaches your laptop's port 3000
```

By default the remote port is bound to the server's loopback only. To bind on all addresses, the server needs `GatewayPorts yes` (or `clientspecified`) in `sshd_config`. Keep that off unless needed.

**Reverse access to a host behind NAT:** the NAT-ed host opens `ssh -N -R 2222:localhost:22 user@public-server`. From the public server, `ssh -p 2222 localhost` reaches the NAT-ed host.

### Dynamic forwarding (-D): SOCKS proxy

```console
$ ssh -N -D 1080 alice@bastion.example.com
$ curl --socks5-hostname 127.0.0.1:1080 http://intranet.example.com/
```

Configure a browser or tool to use the SOCKS5 proxy `127.0.0.1:1080`. `--socks5-hostname` makes the **server** resolve names (avoids DNS leaks).

### Jump hosts (ProxyJump)

```console
$ ssh -J alice@bastion.example.com bob@10.0.0.5           # one hop
$ ssh -J bastion1,bastion2 bob@10.0.0.5                   # several hops
$ scp -J bastion.example.com file.txt bob@10.0.0.5:/tmp/
```

Prefer `ProxyJump` to agent forwarding: your keys stay on your machine, and the final host's authentication happens end to end.

## Commands

### Useful options

| Option | Meaning |
|--------|---------|
| `-N` | Do not run a remote command (tunnel only) |
| `-f` | Go to the background after authentication |
| `-T` | No terminal |
| `-g` | Allow other hosts to connect to your forwarded local ports (**gateway ports**). Avoid |
| `-o ExitOnForwardFailure=yes` | Exit if the tunnel cannot be set up (good for scripts) |
| `-o ServerAliveInterval=30` | Detect dead tunnels |
| `-L 127.0.0.1:5433:...` | Explicitly bind to loopback (the default for `-L`) |
| `-O exit` | Stop a multiplexed master connection |

Keep a background tunnel open:

```console
$ ssh -fN -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -L 5433:db.internal:5432 alice@bastion.example.com
$ ss -tlnp | grep 5433           # verify the local listener
$ pkill -f 'ssh -fN .*5433'      # stop it
```

Configuration in `~/.ssh/config`:

```
Host db-tunnel
    HostName bastion.example.com
    User alice
    LocalForward 5433 db.internal:5432
    ExitOnForwardFailure yes
    ServerAliveInterval 30
```

```console
$ ssh -N db-tunnel
```

### Persistent tunnels with systemd

For tunnels that must survive failures, use a service with automatic restart (or `autossh`):

```ini
# /etc/systemd/system/db-tunnel.service
[Unit]
Description=SSH tunnel to database
After=network-online.target
Wants=network-online.target

[Service]
User=tunnel
ExecStart=/usr/bin/ssh -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -i /etc/tunnel/id_ed25519 -L 127.0.0.1:5433:db.internal:5432 tunnel@bastion.example.com
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Use a dedicated low-privilege account and a restricted key on the bastion (`restrict,port-forwarding,permitopen="db.internal:5432"`).

### Layer 3 tunnels and others

| Feature | Notes |
|---------|-------|
| `ssh -w 0:0` (tun device) | VPN-like layer 3 tunnel. Needs `PermitTunnel` on the server and root. Prefer WireGuard or a real VPN |
| `-X` / `-Y` X11 forwarding | Run graphical programs remotely. Needs `X11Forwarding yes` on the server. `-Y` (trusted) is riskier |
| `sshuttle` | A third-party tool that routes subnets through SSH ("poor man's VPN") |
| `sshfs` | Mount a remote directory over SSH (`sshfs alice@server01:/data /mnt/data`) |

### Control tunnels on the server

```
# sshd_config
AllowTcpForwarding no                # disable all forwarding (default for many hardened setups)
Match User tunnel
    AllowTcpForwarding local
    PermitOpen db.internal:5432
    X11Forwarding no
    PermitTTY no
```

`AllowTcpForwarding` values: `yes`, `no`, `local` (only `-L`), `remote` (only `-R`). `PermitOpen` limits the allowed destinations.

## Common pitfalls

- **Local port already in use.** `bind: Address already in use`. Pick another port or find the process: `ss -tlnp | grep PORT`.
- **Destination confusion.** In `-L`, `DESTHOST` is resolved on the **SSH server**. `localhost` there means the server, not your laptop.
- **Tunnel connects but the application fails.** The service may require TLS or hostname checks that do not match `localhost`. Use `--resolve`-like options, or `/etc/hosts`.
- **`-R` port not reachable from outside.** It binds to loopback unless `GatewayPorts` is enabled.
- **Tunnels dying silently.** Add `ServerAliveInterval`, `ExitOnForwardFailure`, and a supervisor (systemd or autossh).
- **Binding forwarded ports on `0.0.0.0`** (`-g`) exposes the tunnel to the whole network. Keep it on loopback.
- **Bypassing security policy.** Tunnels can hide traffic from firewalls and monitoring. Many organizations restrict them. Follow policy and use `PermitOpen`.
- **Leaving reverse tunnels open** creates a hidden back door. Document and monitor them.
- **Using a SOCKS proxy with DNS leaks.** Use remote DNS resolution (`socks5h://` or `--socks5-hostname`).
- **Privileged local ports** (below 1024) need root. Use a high port.

## Further reading

- [`ssh(1)`: port forwarding](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [`sshd_config(5)`: AllowTcpForwarding, PermitOpen](https://man7.org/linux/man-pages/man5/sshd_config.5.html)
- [autossh](https://www.harding.motd.ca/autossh/)
- [sshuttle](https://github.com/sshuttle/sshuttle)
