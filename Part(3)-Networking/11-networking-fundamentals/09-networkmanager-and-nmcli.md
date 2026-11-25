# NetworkManager and nmcli

## Overview

**NetworkManager** is the network service on RHEL-based systems, Fedora, SUSE desktops, Ubuntu desktop, and many servers. It manages **connection profiles** (saved settings) for Ethernet, Wi-Fi, VPNs, bonds, bridges, and VLANs. The command-line tool is **`nmcli`**. A text user interface, **`nmtui`**, is also available.

## Key concepts

### Devices and connections

| Term | Meaning |
|------|---------|
| **Device** | A network interface (`enp3s0`, `wlp2s0`) |
| **Connection (profile)** | A named set of settings: address method, IP, DNS, security. Can be applied to a device |
| **Active connection** | A profile currently applied to a device |

One device can have several profiles (for example, "Office" and "Lab"), but only one is active at a time.

### Settings use `group.property` names

| Property | Meaning |
|----------|---------|
| `connection.id` | Profile name |
| `connection.interface-name` | Device the profile applies to |
| `connection.autoconnect` | Activate at boot (`yes`/`no`) |
| `ipv4.method` | `auto` (DHCP), `manual` (static), `link-local`, `disabled`, `shared` |
| `ipv4.addresses` | `192.0.2.10/24` |
| `ipv4.gateway` | `192.0.2.1` |
| `ipv4.dns` | `192.0.2.53 198.51.100.53` |
| `ipv4.dns-search` | `example.com` |
| `ipv4.routes` | `10.20.0.0/16 192.0.2.254` |
| `ipv6.method` | `auto`, `dhcp`, `manual`, `ignore`, `disabled` |
| `802-3-ethernet.mtu` | MTU |

## Commands

### Overview

```console
$ nmcli general status
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
connected  full          enabled  enabled  missing  enabled

$ nmcli device status
DEVICE   TYPE      STATE      CONNECTION
enp3s0   ethernet  connected  Wired connection 1
wlp2s0   wifi      disconnected  --
lo       loopback  unmanaged  --

$ nmcli connection show
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  3f1c9a1e-8d5e-4a3b-9f11-0d8d6d5b1c22  ethernet  enp3s0

$ nmcli connection show "Wired connection 1"       # all settings (long)
$ nmcli -f ipv4 connection show "Wired connection 1"
$ nmcli device show enp3s0                         # runtime values (IP, gateway, DNS)
```

### Static IP configuration

Change an existing profile:

```console
# nmcli connection modify "Wired connection 1" \
      ipv4.method manual \
      ipv4.addresses 192.0.2.10/24 \
      ipv4.gateway 192.0.2.1 \
      ipv4.dns "192.0.2.53 198.51.100.53" \
      ipv4.dns-search example.com
# nmcli connection up "Wired connection 1"
```

Create a new profile:

```console
# nmcli connection add type ethernet con-name lan0 ifname enp3s0 \
      ipv4.method manual ipv4.addresses 192.0.2.10/24 ipv4.gateway 192.0.2.1 \
      ipv4.dns 192.0.2.53 connection.autoconnect yes
# nmcli connection up lan0
```

Switch back to DHCP:

```console
# nmcli connection modify lan0 ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
# nmcli connection up lan0
```

> [!IMPORTANT]
> `nmcli connection modify` **saves** the change but does **not** apply it. Run `nmcli connection up NAME` (or `nmcli device reapply DEVICE`) to apply it. Applying on a remote system can disconnect you. Be careful.

### Add or remove values: `+` and `-`

```console
# nmcli connection modify lan0 +ipv4.addresses 192.0.2.11/24          # add a second address
# nmcli connection modify lan0 -ipv4.addresses 192.0.2.11/24          # remove it
# nmcli connection modify lan0 +ipv4.routes "10.20.0.0/16 192.0.2.254"   # add a static route
# nmcli connection modify lan0 +ipv4.dns 198.51.100.53
# nmcli connection modify lan0 ipv4.route-metric 100
```

### Activate, deactivate, remove

| Command | Purpose |
|---------|---------|
| `nmcli connection up NAME` | Activate a profile |
| `nmcli connection down NAME` | Deactivate |
| `nmcli connection delete NAME` | Delete a profile |
| `nmcli connection reload` | Re-read profile files from disk |
| `nmcli device connect enp3s0` | Connect a device using a suitable profile |
| `nmcli device disconnect enp3s0` | Disconnect (does not auto-reconnect until you connect again) |
| `nmcli device reapply enp3s0` | Apply modified settings to the active connection without a full disconnect |
| `nmcli device modify enp3s0 ipv4.dns 192.0.2.53` | **Temporary** change (not saved) |
| `nmcli networking off` / `on` | Disable or enable all networking |
| `nmcli device set enp3s0 managed no` | Stop NetworkManager from managing a device |

### Wi-Fi

```console
$ nmcli device wifi list
$ nmcli device wifi connect "MySSID" password "secret"
$ nmcli radio wifi off
```

Avoid passwords on the command line in shared environments (history and process list). Use `--ask` to be prompted: `nmcli --ask device wifi connect MySSID`.

### Bond, bridge, and VLAN examples

```console
# nmcli connection add type bridge con-name br0 ifname br0 ipv4.method manual ipv4.addresses 192.0.2.10/24 ipv4.gateway 192.0.2.1
# nmcli connection add type bridge-slave con-name br0-port1 ifname enp3s0 master br0

# nmcli connection add type bond con-name bond0 ifname bond0 bond.options "mode=802.3ad,miimon=100"
# nmcli connection add type ethernet slave-type bond con-name bond0-p1 ifname enp3s0 master bond0
# nmcli connection add type ethernet slave-type bond con-name bond0-p2 ifname enp4s0 master bond0

# nmcli connection add type vlan con-name vlan100 ifname enp3s0.100 dev enp3s0 id 100 ipv4.method manual ipv4.addresses 198.51.100.10/24
```

### Text interfaces

```console
# nmtui                       # menu-driven editor (package NetworkManager-tui)
$ nmcli -p connection show lan0       # pretty print
$ nmcli -t -f NAME,DEVICE connection show --active     # terse, for scripts
$ nmcli monitor               # watch events
```

## Configuration files

| Path | Purpose |
|------|---------|
| `/etc/NetworkManager/NetworkManager.conf` | Main configuration |
| `/etc/NetworkManager/conf.d/*.conf` | Drop-in configuration |
| `/etc/NetworkManager/system-connections/*.nmconnection` | **Profiles** (keyfile format). Permissions must be `0600`, owned by root |
| `/etc/sysconfig/network-scripts/ifcfg-*` | Legacy "ifcfg" profiles (older RHEL-based). Deprecated |
| `/run/NetworkManager/` | Runtime data and generated profiles |
| `/var/lib/NetworkManager/` | Leases and state |

Example keyfile `/etc/NetworkManager/system-connections/lan0.nmconnection`:

```ini
[connection]
id=lan0
type=ethernet
interface-name=enp3s0
autoconnect=true

[ipv4]
method=manual
address1=192.0.2.10/24,192.0.2.1
dns=192.0.2.53;198.51.100.53;
dns-search=example.com;

[ipv6]
method=auto
```

After editing a file: `chmod 600 file && nmcli connection reload && nmcli connection up lan0`.

### Which distributions use it

| Distribution | Default |
|--------------|---------|
| RHEL-based | NetworkManager (servers and desktops) |
| Fedora | NetworkManager |
| Ubuntu Desktop | NetworkManager. **Ubuntu Server**: netplan with systemd-networkd (NetworkManager is possible) |
| Debian | Depends on install type. Desktop uses NetworkManager. Server often `ifupdown` |
| SUSE | Desktop: NetworkManager. SLES 15: wicked by default (newer releases move toward NetworkManager). Check your version |

Run **one** network manager per interface. Check with `systemctl status NetworkManager systemd-networkd wicked`.

## Common pitfalls

- **Modifying a profile and not applying it.** Run `nmcli connection up NAME` or `nmcli device reapply`.
- **Using the wrong profile name.** Names with spaces need quotes. List with `nmcli connection show`.
- **Several profiles for one device with `autoconnect`.** NetworkManager picks by priority (`connection.autoconnect-priority`). Delete unused profiles.
- **Device marked `unmanaged`.** NetworkManager ignores it (often another tool manages it, or `managed=false`). Check `nmcli device`.
- **Static IP without `ipv4.method manual`.** The address is ignored or mixed with DHCP.
- **Leftover DNS from DHCP.** Set `ipv4.ignore-auto-dns yes` to use only your own DNS.
- **Keyfile permissions wrong.** NetworkManager ignores files that are not `0600` owned by root.
- **Remote changes locking you out.** Use the console, or a rollback plan (a timer that reverts, such as `at now + 5 min` with a restore command).
- **Docker, libvirt, and Kubernetes interfaces** are usually set unmanaged in NetworkManager. Avoid having it manage them.

## Further reading

- [`nmcli(1)`](https://man7.org/linux/man-pages/man1/nmcli.1.html)
- [`nm-settings-nmcli(5)`](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html)
- [NetworkManager documentation](https://networkmanager.dev/docs/)
- [Red Hat: configuring and managing networking](https://docs.redhat.com)
