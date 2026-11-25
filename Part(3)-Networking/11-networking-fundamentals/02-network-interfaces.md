# Network Interfaces

## Overview

A **network interface** is the connection point between a Linux system and a network. It may be a real network card, a wireless adapter, or a **virtual** interface created by software (bridges, VLANs, containers, VPNs). Every IP address belongs to an interface.

## Key concepts

### Interface names

Modern systems use **predictable names**, assigned by udev (`systemd`), that stay the same across reboots.

| Pattern | Meaning | Example |
|---------|---------|---------|
| `eno1` | Onboard, index from firmware | `eno1` |
| `ens3` | PCI hotplug slot | `ens18` (common on VMs) |
| `enp3s0` | PCI bus 3, slot 0 | `enp3s0` |
| `wlp2s0` | Wireless | `wlp2s0` |
| `eth0` | **Legacy** kernel name. Still used in some clouds, containers, and with `net.ifnames=0` | `eth0` |
| `lo` | Loopback | `lo` |

`en` = Ethernet, `wl` = wireless LAN, `ww` = wireless WAN.

### Interface types

| Type | Purpose |
|------|---------|
| **Physical** | Real hardware (Ethernet, Wi-Fi) |
| **Loopback** (`lo`) | Local-only interface for the host itself (`127.0.0.1`, `::1`) |
| **Bridge** (`br0`) | Connects interfaces like a virtual switch (VMs, containers) |
| **Bond** (`bond0`) | Combines several NICs for redundancy or throughput |
| **VLAN** (`eth0.100`) | Tagged virtual network on one NIC |
| **veth** | Virtual cable pair, used by containers |
| **tun / tap** | Software interfaces for VPNs and VMs |
| **WireGuard** (`wg0`) | VPN tunnel interface |
| **dummy** | Fake interface for testing or holding addresses |

### Link states

In `ip link`, flags show the state:

| Flag | Meaning |
|------|---------|
| `UP` | Administratively enabled |
| `LOWER_UP` | Physical link detected (cable plugged in, carrier present) |
| `NO-CARRIER` | Enabled, but no link (cable unplugged, switch port down) |
| `state DOWN` | Disabled |
| `BROADCAST,MULTICAST` | Capabilities |
| `mtu 1500` | Maximum packet size |

`UP` without `LOWER_UP` means "turned on but no signal".

## Commands

### Show interfaces

```console
$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp3s0           UP             52:54:00:12:34:56 <BROADCAST,MULTICAST,UP,LOWER_UP>
wlp2s0           DOWN           a4:c3:f0:11:22:33 <BROADCAST,MULTICAST>

$ ip -br addr
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp3s0           UP             192.0.2.10/24 fe80::5054:ff:fe12:3456/64

$ ip -s link show enp3s0          # with statistics
    RX: bytes  packets  errors  dropped  missed  mcast
    9812345678 8123456  0       12       0       4521
    TX: bytes  packets  errors  dropped  carrier collsns
    1234567890 4123456  0       0        0       0
```

Watch `errors`, `dropped`, and `carrier` counters. Rising values point to cable, driver, or capacity problems.

### Change state and settings (root, temporary)

```console
# ip link set enp3s0 down
# ip link set enp3s0 up
# ip link set enp3s0 mtu 9000                # jumbo frames (all devices on the path must support them)
# ip link set enp3s0 address 52:54:00:aa:bb:cc   # change MAC (link must be down)
# ip link set enp3s0 promisc on              # promiscuous mode (packet capture)
```

These changes are lost at reboot. See [Network Configuration Persistence](11-network-configuration-persistence.md).

> [!WARNING]
> On a remote server, taking down the interface you are connected through ends your session. Use the console or a second interface.

### ethtool: physical link details

```console
# ethtool enp3s0
Settings for enp3s0:
        Speed: 1000Mb/s
        Duplex: Full
        Auto-negotiation: on
        Link detected: yes

# ethtool -i enp3s0                 # driver and firmware
driver: e1000e
version: 6.8.0
firmware-version: 0.5-4

# ethtool -S enp3s0 | grep -i -E 'err|drop|crc'     # NIC statistics
# ethtool -k enp3s0                 # offload features (TSO, GRO, checksum)
```

| Check | Healthy value |
|-------|---------------|
| `Speed` / `Duplex` | Expected speed, **Full** duplex |
| `Link detected` | `yes` |
| CRC / error counters | Zero or not growing |

### Other useful commands

| Command | Purpose |
|---------|---------|
| `ls /sys/class/net/` | List interfaces |
| `cat /sys/class/net/enp3s0/operstate` | Link state (`up`, `down`) |
| `cat /sys/class/net/enp3s0/speed` | Speed in Mbit/s |
| `nmcli device status` | NetworkManager's view |
| `networkctl list` | systemd-networkd's view |
| `lspci \| grep -i ethernet` | Find the hardware |
| `dmesg \| grep -i -E 'enp3s0\|link is'` | Link up/down messages |
| `ip -d link show br0` | Details (bridge, VLAN, bond options) |

### Create virtual interfaces (examples, temporary)

```console
# ip link add br0 type bridge                       # bridge
# ip link set enp3s0 master br0
# ip link add link enp3s0 name enp3s0.100 type vlan id 100     # VLAN 100
# ip link add dummy0 type dummy
# ip link add veth0 type veth peer name veth1       # virtual cable
# ip link add bond0 type bond mode 802.3ad          # LACP bond
# ip link del dummy0                                # delete
```

For permanent setups use your network manager (see [nmcli](09-networkmanager-and-nmcli.md) and [Persistence](11-network-configuration-persistence.md)).

### Bonding modes (overview)

| Mode | Name | Notes |
|------|------|-------|
| 0 | balance-rr | Round robin. Can reorder packets |
| 1 | active-backup | One active NIC, one standby. No switch support needed |
| 2 | balance-xor | Hash-based |
| 4 | 802.3ad (LACP) | Switch must support and be configured for LACP. Best for throughput and redundancy |
| 5, 6 | balance-tlb, balance-alb | Adaptive load balancing, no special switch support |

## Common pitfalls

- **Wrong interface name in a copied configuration.** Check `ip -br link`. Names differ between systems (`ens3`, `enp3s0`, `eth0`).
- **Interface names changed after hardware or kernel changes** or after adding PCI devices. Pin names with systemd `.link` files or use MAC-based matching in your configuration.
- **`UP` but no connectivity.** Check `LOWER_UP`, `ethtool`, the cable, the switch port, VLAN, and `NO-CARRIER`.
- **Speed/duplex mismatch.** Half duplex or 100 Mb/s on a gigabit link causes slowness and errors. Check `ethtool` and the switch.
- **Jumbo frames set on one side only.** Large packets are dropped. All devices in the path need the same MTU.
- **Offload artifacts in packet captures.** Checksums look wrong in local captures because the NIC calculates them later. Do not treat that as an error.
- **Forgetting the loopback.** Services bound to `127.0.0.1` are reachable only locally.

## Further reading

- [`ip-link(8)`](https://man7.org/linux/man-pages/man8/ip-link.8.html)
- [`ethtool(8)`](https://man7.org/linux/man-pages/man8/ethtool.8.html)
- [Predictable network interface names (systemd)](https://www.freedesktop.org/software/systemd/man/latest/systemd.net-naming-scheme.html)
- [Kernel bonding documentation](https://www.kernel.org/doc/html/latest/networking/bonding.html)
