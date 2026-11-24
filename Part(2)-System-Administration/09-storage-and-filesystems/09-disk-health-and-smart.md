# Disk Health and SMART Monitoring

## Overview

Disks usually give warning signs before they fail. **SMART** (Self-Monitoring, Analysis and Reporting Technology) is a feature inside HDDs, SSDs, and NVMe drives that records health counters and error logs. The `smartmontools` package reads them and can alert you.

> [!NOTE]
> SMART helps but is **not a guarantee**. Many disks fail with no SMART warning, and a "PASSED" result does not mean the disk is healthy. Always keep backups and redundancy.

## Key concepts

- **SATA/SAS disks** report a table of numbered **attributes**.
- **NVMe disks** report a standard **health log** (spare capacity, wear, media errors).
- Virtual disks (VMs, cloud volumes) have no SMART data. Monitor at the host or cloud level.
- Disks behind **hardware RAID controllers** need a controller-specific option, for example `smartctl -d megaraid,0 /dev/sda`.

### Attributes that matter (SATA/SAS)

| Attribute | Meaning | Concern when |
|-----------|---------|--------------|
| `Reallocated_Sector_Ct` (5) | Bad sectors replaced by spares | Above 0 and growing |
| `Current_Pending_Sector` (197) | Sectors waiting to be reallocated (unreadable) | Above 0 |
| `Offline_Uncorrectable` (198) | Sectors that could not be corrected | Above 0 |
| `Reported_Uncorrect` (187) | Uncorrectable read errors | Above 0 |
| `UDMA_CRC_Error_Count` (199) | Errors on the cable or connection | Growing: replace cable or check the port |
| `Temperature_Celsius` (194) | Temperature | Constantly above about 50–55 °C for HDDs |
| `Power_On_Hours` (9) | Age | Informational |
| `Wear_Leveling_Count`, `Media_Wearout_Indicator`, `SSD_Life_Left` | SSD wear | Near the vendor's end-of-life value |

### Attributes that matter (NVMe)

| Field | Meaning |
|-------|---------|
| `Critical Warning` | Non-zero means the drive reports a problem |
| `Available Spare` and `Available Spare Threshold` | Remaining spare blocks; concern if near the threshold |
| `Percentage Used` | Wear (100% = rated endurance reached; drives often continue longer) |
| `Media and Data Integrity Errors` | Should be 0 |
| `Temperature` | Check against the vendor's limits |

## Commands

Install: **Debian-based** `apt install smartmontools`, **RHEL-based** `dnf install smartmontools`, **SUSE-based** `zypper install smartmontools`. NVMe: `nvme-cli`.

### Check health (root)

```console
# smartctl -H /dev/sda                       # overall result
SMART overall-health self-assessment test result: PASSED

# smartctl -i /dev/sda                       # model, serial, capabilities
# smartctl -A /dev/sda                       # attributes only
# smartctl -a /dev/sda                       # all information
# smartctl -x /dev/sda                       # extended
# smartctl -a /dev/nvme0                     # NVMe
# nvme smart-log /dev/nvme0
```

```console
# smartctl -A /dev/sda
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  5 Reallocated_Sector_Ct   0x0033   100   100   010    Pre-fail  Always       -       0
  9 Power_On_Hours          0x0032   088   088   000    Old_age   Always       -       52310
197 Current_Pending_Sector  0x0032   100   100   000    Old_age   Always       -       0
199 UDMA_CRC_Error_Count    0x003e   200   200   000    Old_age   Always       -       0
```

Read it as: normalized `VALUE` falls toward `THRESH`; the **`RAW_VALUE`** of the error counters is what you watch.

### Self-tests

Tests run inside the drive. Short tests take minutes, long tests take hours.

```console
# smartctl -t short /dev/sda                 # quick test
# smartctl -t long /dev/sda                  # full surface test (may affect performance)
# smartctl -l selftest /dev/sda              # results
# smartctl -l error /dev/sda                 # error log
# smartctl -c /dev/sda                       # capabilities and test duration
```

### Monitor continuously: smartd

`smartd` checks drives regularly and can send mail or run a script.

```
# /etc/smartd.conf
DEVICESCAN -a -o on -S on -n standby,q -s (S/../.././02|L/../../6/03) -W 4,45,55 -m root
```

Meaning: monitor all devices, run a short test daily at 02:00 and a long test on Saturdays at 03:00, warn on temperature changes, and mail `root`.

```console
# systemctl enable --now smartd              # Debian-based service name: smartmontools (or smartd)
# systemctl status smartd
```

Make sure mail to `root` is delivered to a person or monitoring system.

### Related checks

| Task | Command |
|------|---------|
| Kernel I/O errors | `dmesg -T \| grep -i -E 'I/O error\|ata[0-9]\|nvme\|medium error'` |
| Journal errors | `journalctl -k -p err` |
| RAID state | `cat /proc/mdstat` |
| Filesystem errors | `dmesg \| grep -i -E 'EXT4-fs error\|XFS.*error'` |
| Disk speed sanity test | `hdparm -tT /dev/sda` |
| Surface scan (read-only) | `badblocks -sv /dev/sdb` |

> [!WARNING]
> `badblocks -w` (write mode) **destroys all data**. Use the default read-only test, or only use write mode on empty disks.

### When a disk looks bad

1. **Back up** or copy data off immediately (`rsync`; for a failing disk use `ddrescue` instead of `dd`).
2. Check redundancy (RAID, replicas) and start replacement.
3. Replace the disk. Do not wait for complete failure.
4. After replacement, rebuild and verify the array and filesystem.

```console
# ddrescue -f -n /dev/sdb /dev/sdc rescue.log     # copy a failing disk to a new one (all data on sdc is replaced)
```

## Common pitfalls

- **Trusting "PASSED".** Look at the raw attribute values and error logs.
- **Ignoring `Current_Pending_Sector`.** A non-zero value means data may already be unreadable.
- **Running SMART on virtual disks** and finding nothing. Monitor the underlying platform.
- **Not testing alerting.** A configured `smartd` that mails to an unread mailbox is useless.
- **Disks behind RAID controllers** need `-d TYPE` options.
- **Running a long test on a busy production disk** can add latency. Schedule off-peak.
- **Treating cable errors as disk failure.** Rising `UDMA_CRC_Error_Count` points to the cable or backplane.
- **Waiting for failure.** Replace disks at the first sign of growing reallocated or pending sectors in a server.

## Further reading

- [smartmontools](https://www.smartmontools.org)
- [`smartctl(8)`](https://man7.org/linux/man-pages/man8/smartctl.8.html)
- [`smartd.conf(5)`](https://man7.org/linux/man-pages/man5/smartd.conf.5.html)
- [nvme-cli](https://github.com/linux-nvme/nvme-cli)
