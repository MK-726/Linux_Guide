# Network Configuration Persistence

## Overview

Commands such as `ip addr add` and `ip route add` change the **running** system only. After a reboot or a service restart they are gone. To make a setting permanent, you must put it in the configuration system your distribution uses. Different distributions use different tools, and using the wrong one is a common source of "it worked until reboot" problems.

## Key concepts

### Runtime vs. persistent

| Type | Example | Survives reboot? |
|------|---------|------------------|
| **Runtime** | `ip addr add ...`, `nmcli device modify ...`, `resolvectl dns ...` | **No** |
| **Persistent** | Netplan YAML, NetworkManager profile, `.network` file, `/etc/network/interfaces` | **Yes** |

### Who configures the network?

| Distribution | Typical tool | Configuration location |
|--------------|-------------|------------------------|
| **Ubuntu Server** | **Netplan** (renders to systemd-networkd or NetworkManager) | `/etc/netplan/*.yaml` |
| **Ubuntu Desktop** | NetworkManager (through netplan) | `/etc/netplan/`, `/etc/NetworkManager/` |
| **Debian** | `ifupdown` (server) or NetworkManager (desktop), or systemd-networkd | `/etc/network/interfaces`, `/etc/network/interfaces.d/` |
| **RHEL-based** | **NetworkManager** | `/etc/NetworkManager/system-connections/*.nmconnection` (keyfile). Older: `/etc/sysconfig/network-scripts/ifcfg-*` |
| **SUSE-based** | **wicked** (SLES 15, traditional) or NetworkManager | `/etc/sysconfig/network/ifcfg-*` (wicked), NetworkManager files |
| **Any systemd system** | **systemd-networkd** (optional) | `/etc/systemd/network/*.network`, `*.netdev`, `*.link` |
| **Cloud images** | `cloud-init` generates the configuration at boot | `/etc/cloud/`, rendered into netplan/NM/ifcfg |

> [!IMPORTANT]
> Use **one** tool per interface. Check what is running: `systemctl is-active NetworkManager systemd-networkd wicked networking`, and `nmcli device` / `networkctl list`.

## Commands and examples

### Netplan (Ubuntu)

`/etc/netplan/01-netcfg.yaml` (YAML uses **spaces**, not tabs):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
      addresses:
        - 192.0.2.10/24
        - 2001:db8:10:1::10/64
      routes:
        - to: default
          via: 192.0.2.1
        - to: 10.20.0.0/16
          via: 192.0.2.254
      nameservers:
        search: [example.com]
        addresses: [192.0.2.53, 198.51.100.53]
    enp4s0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
```

```console
# chmod 600 /etc/netplan/*.yaml        # netplan warns if files are readable by others
# netplan generate                     # validate and create backend configuration
# netplan try                          # apply, and ROLL BACK after 120 s unless you press Enter
# netplan apply                        # apply permanently
$ netplan get                          # show the merged configuration
$ netplan status                       # (newer versions) show state
```

> [!TIP]
> **Always use `netplan try` on remote systems.** If the new configuration breaks your connection, it reverts automatically.

Old style `gateway4:` is deprecated. Use `routes: - to: default`.

### Debian: ifupdown

`/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

auto enp3s0
iface enp3s0 inet static
    address 192.0.2.10/24
    gateway 192.0.2.1
    dns-nameservers 192.0.2.53 198.51.100.53
    dns-search example.com

auto enp4s0
iface enp4s0 inet dhcp

iface enp3s0 inet6 auto
```

```console
# ifup enp3s0 / ifdown enp3s0
# systemctl restart networking
```

`dns-nameservers` requires the `resolvconf` package (or another resolver integration) to take effect.

### RHEL-based and NetworkManager

Create or modify profiles with `nmcli` (see [NetworkManager and nmcli](09-networkmanager-and-nmcli.md)). They are saved as keyfiles automatically. Check:

```console
$ ls -l /etc/NetworkManager/system-connections/
-rw------- 1 root root 312 Mar  3 10:00 lan0.nmconnection
```

Legacy `ifcfg` file (older RHEL 7/8 style) for reference:

```
# /etc/sysconfig/network-scripts/ifcfg-enp3s0
TYPE=Ethernet
BOOTPROTO=none
NAME=enp3s0
DEVICE=enp3s0
ONBOOT=yes
IPADDR=192.0.2.10
PREFIX=24
GATEWAY=192.0.2.1
DNS1=192.0.2.53
```

> [!NOTE]
> The `ifcfg` format is deprecated on newer RHEL releases in favor of keyfiles. Migrate with `nmcli connection migrate` (where available).

### SUSE: wicked

```
# /etc/sysconfig/network/ifcfg-eth0
BOOTPROTO='static'
STARTMODE='auto'
IPADDR='192.0.2.10/24'
```

```
# /etc/sysconfig/network/routes
default 192.0.2.1 - -
```

```console
# wicked ifreload all
# wicked ifstatus all
```

### systemd-networkd (any distribution)

`/etc/systemd/network/20-wired.network`:

```ini
[Match]
Name=enp3s0

[Network]
Address=192.0.2.10/24
Gateway=192.0.2.1
DNS=192.0.2.53
DNS=198.51.100.53
Domains=example.com

[Route]
Destination=10.20.0.0/16
Gateway=192.0.2.254
```

```console
# systemctl enable --now systemd-networkd
# networkctl reload
$ networkctl status enp3s0
$ networkctl list
```

A bridge needs a `.netdev` file (`Kind=bridge`) plus `.network` files. Match interfaces by **MAC address** (`MACAddress=`) when names may change.

### Persistent routes

| Tool | How |
|------|-----|
| Netplan | `routes:` list |
| NetworkManager | `nmcli con mod NAME +ipv4.routes "10.20.0.0/16 192.0.2.254"` |
| ifupdown | `up ip route add 10.20.0.0/16 via 192.0.2.254` under the interface |
| networkd | `[Route]` section |
| wicked | `/etc/sysconfig/network/routes` |

### Persistent kernel settings

```console
# echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-forward.conf
# sysctl --system
```

### Persistent interface names

Pin a name to a MAC address with a systemd `.link` file:

```ini
# /etc/systemd/network/10-lan0.link
[Match]
MACAddress=52:54:00:12:34:56

[Link]
Name=lan0
```

Rebuild the initramfs afterwards if the interface is used early in boot.

### Safe change procedure for remote systems

1. Make sure you have **console or out-of-band access**.
2. Back up the current configuration (`cp -a`, or `netplan get > backup.yaml`).
3. Use a rollback method: `netplan try`, or schedule a revert: `echo "cp /root/backup.yaml /etc/netplan/01.yaml && netplan apply" | at now + 5 minutes` and remove the job (`atrm`) when everything works.
4. Apply the change and **test from a second terminal** before closing the first.
5. Verify after a reboot (when possible).

## Common pitfalls

- **Using `ip` commands and expecting them to persist.** Put the setting in the correct configuration system.
- **Two managers fighting** (NetworkManager and systemd-networkd, or ifupdown and netplan). One overwrites the other. Disable all but one for each interface.
- **YAML indentation errors** in Netplan (tabs, wrong nesting). Run `netplan generate` to see errors.
- **Wrong file permissions** on netplan or NetworkManager files. They are ignored or produce warnings.
- **cloud-init overwrites your changes at the next boot.** Disable its network management: create `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` with `network: {config: disabled}`.
- **Interface names differ between machines.** Copying configuration between hosts needs name changes (or matching by MAC).
- **DNS set in two places** (DHCP and static) leads to unexpected servers. Check `resolvectl status` or `nmcli device show`.
- **Forgetting IPv6.** Disable or configure it deliberately.
- **Applying without a way back.** Always plan a rollback for remote machines.
- **Editing `/etc/resolv.conf` or `/etc/hosts` only** to change the address of the host. Those files do not set the address.

## Further reading

- [Netplan documentation](https://netplan.readthedocs.io)
- [`systemd.network(5)`](https://man7.org/linux/man-pages/man5/systemd.network.5.html)
- [`interfaces(5)`](https://manpages.debian.org/stable/ifupdown/interfaces.5.en.html)
- [NetworkManager keyfile format](https://networkmanager.dev/docs/api/latest/nm-settings-keyfile.html)
- [cloud-init networking](https://cloudinit.readthedocs.io/en/latest/reference/network-config.html)
