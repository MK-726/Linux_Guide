# Time Zones, System Clocks, NTP, and chrony

## Overview

Correct time matters more than most people expect. Wrong clocks break **TLS certificate checks**, **Kerberos / Active Directory logins**, **log correlation**, **scheduled jobs**, **database replication**, and **security investigations**. Every server should keep accurate time automatically.

## Key concepts

### Two clocks

| Clock | Description |
|-------|-------------|
| **System clock** | Kept by the kernel while the system runs. This is what `date` shows |
| **Hardware clock (RTC)** | A battery-backed clock on the motherboard. Used to set the system clock at boot |

**Recommendation:** keep the hardware clock in **UTC** and set the time zone in software. Windows defaults to local time in the RTC, which causes problems on dual-boot systems.

### Time zones

A time zone is a rule set. Files in `/usr/share/zoneinfo/` (the **tz database**) describe them, and `/etc/localtime` is a link to the chosen one. Time zone names use `Region/City`: `Europe/Berlin`, `America/New_York`, `UTC`.

**For servers, use UTC** unless there is a strong reason not to. It avoids daylight-saving confusion in logs and schedules.

### NTP and synchronization

**NTP** (Network Time Protocol) synchronizes clocks over the network (UDP port 123). Terms:

| Term | Meaning |
|------|---------|
| **Stratum** | Distance from a reference clock. Stratum 0 = atomic/GPS clock. Stratum 1 is directly connected to it. Each hop adds 1 |
| **Offset** | Difference between your clock and the reference |
| **Step** | A sudden jump to the correct time (used at start or for big errors) |
| **Slew** | Gradually speeding up or slowing the clock to correct it (preferred while running; avoids time going backwards) |
| **Pool** | A DNS name for many volunteer servers (`pool.ntp.org`) |
| **NTS** | Network Time Security: authenticated NTP (supported by chrony) |

### Time synchronization software

| Software | Notes |
|----------|-------|
| **chrony** | Modern, accurate, fast convergence, good for VMs and laptops. **Default on RHEL-based and SUSE-based**; available everywhere |
| **systemd-timesyncd** | Simple SNTP client. Often the default on Debian-based systems. Good for basic needs, not a full NTP server |
| **ntpd** | The classic reference implementation. Older systems and special setups |
| **PTP (`linuxptp`)** | High-precision time over a local network (finance, telecom) |

Check what your system uses: `systemctl status chronyd systemd-timesyncd ntp` (service name `chrony` on Debian-based).

## Commands

### timedatectl

```console
$ timedatectl
               Local time: Mon 2025-03-03 11:30:02 UTC
           Universal time: Mon 2025-03-03 11:30:02 UTC
                 RTC time: Mon 2025-03-03 11:30:02
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

| Command | Purpose |
|---------|---------|
| `timedatectl list-timezones \| grep Berlin` | Find a zone |
| `timedatectl set-timezone Europe/Berlin` | Set the time zone (**root**) |
| `timedatectl set-timezone UTC` | Use UTC |
| `timedatectl set-ntp true` | Enable time synchronization |
| `timedatectl set-time "2025-03-03 11:30:00"` | Set the time by hand (disables NTP first) |
| `timedatectl timesync-status` | Status when using systemd-timesyncd |
| `date` / `date -u` | Local / UTC time |
| `date +%s` | Seconds since 1970 (epoch) |
| `TZ=Asia/Tokyo date` | Show time in another zone for one command |
| `hwclock --show` | Read the hardware clock (**root**) |
| `hwclock --systohc` | Copy system time to the hardware clock |

Look for `System clock synchronized: yes` and `NTP service: active`.

### chrony

Install and enable:

```console
# dnf install chrony && systemctl enable --now chronyd          # RHEL-based
# apt install chrony && systemctl enable --now chrony           # Debian-based
```

Configuration file: `/etc/chrony.conf` (RHEL-based, SUSE-based) or `/etc/chrony/chrony.conf` (Debian-based).

```
# Time sources
pool 2.pool.ntp.org iburst
server ntp1.example.com iburst prefer

# Step the clock if off by more than 1 second during the first 3 updates
makestep 1.0 3

# Keep the hardware clock in sync
rtcsync

# Where to store drift information
driftfile /var/lib/chrony/drift

# Allow clients in your network to use this server (only if this is an NTP server)
#allow 192.168.10.0/24
```

| Directive | Meaning |
|-----------|---------|
| `pool NAME iburst` | Use several servers from a pool. `iburst` speeds up the first sync |
| `server NAME iburst` | One specific server |
| `makestep 1.0 3` | Allow stepping for large errors in the first updates |
| `rtcsync` | Update the RTC from the system clock |
| `allow NETWORK` | Act as an NTP server for these clients |
| `nts` (on a `server` line) | Use authenticated NTP |

```console
# systemctl restart chronyd
$ chronyc tracking
Reference ID    : A29FC801 (time.example.net)
Stratum         : 3
System time     : 0.000012345 seconds fast of NTP time
Last offset     : +0.000008123 seconds
RMS offset      : 0.000021 seconds
Leap status     : Normal

$ chronyc sources -v          # list sources. '*' marks the selected one
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* time.example.net              2   6   377    21   +12us[  +25us] +/-  15ms
^+ ntp2.example.org              2   6   377    22   -45us[  -32us] +/-  21ms

$ chronyc sourcestats
$ chronyc clients             # (on a server, root) who uses this server
# chronyc makestep            # force an immediate correction
```

In `sources`, `^*` = current best source, `^+` = acceptable, `^?` = unreachable or unusable. `Reach 377` (octal) means the last 8 polls all succeeded.

### Firewall

Allow **outbound UDP 123** to your time sources. An NTP server also needs **inbound UDP 123** from clients.

### Virtual machines and containers

- **VMs** should still run chrony. Also consider the hypervisor's time sync (for example, the PTP clock device `/dev/ptp_kvm`, or `refclock PHC /dev/ptp0` in chrony) for the best accuracy. Avoid configuring two competing mechanisms.
- **Containers** use the **host's clock**. They cannot set their own time, so fix the host.
- **Cloud providers** offer internal NTP endpoints (for example, AWS provides one at `169.254.169.123`). Use your provider's recommendation.

### Change time zone for a single application

Applications read the `TZ` environment variable, or in systemd: `Environment=TZ=UTC` in the unit file.

## Common pitfalls

- **Setting the time by hand while NTP is active.** NTP overwrites it. Disable NTP first (`timedatectl set-ntp false`) only for testing.
- **Running two NTP services** (for example, chrony and systemd-timesyncd) at once. Only one should be enabled. Installing chrony normally disables timesyncd.
- **Firewall blocks UDP 123.** `chronyc sources` shows `Reach 0`.
- **Big jumps in time** break running software (databases, schedulers). Use slewing. Avoid stepping while production workloads run.
- **Wrong time zone in the application, not the OS.** Check Java (`-Duser.timezone`), PHP, and database settings.
- **Daylight saving time** moves local-time cron jobs. Jobs at 02:30 can be skipped or run twice. Use UTC or avoid those hours.
- **Dual-boot with Windows:** RTC in local time vs UTC. `timedatectl set-local-rtc 1` is a compatibility option, not recommended otherwise.
- **VM paused or restored from snapshot** has a wrong clock until it resynchronizes. Check after restore.
- **Leap seconds.** Modern chrony and kernels can smear or step. Follow your platform's policy and keep all hosts consistent.

## Further reading

- [`timedatectl(1)`](https://man7.org/linux/man-pages/man1/timedatectl.1.html)
- [chrony documentation](https://chrony-project.org/documentation.html)
- [`chrony.conf(5)`](https://chrony-project.org/doc/4.5/chrony.conf.html)
- [NTP Pool Project](https://www.ntppool.org)
- [IANA Time Zone Database](https://www.iana.org/time-zones)
