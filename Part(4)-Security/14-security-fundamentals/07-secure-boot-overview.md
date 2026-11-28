# Secure Boot Overview

## Overview

If an attacker can change what runs **before** the operating system starts (the bootloader, kernel, or initramfs), all later security can be bypassed. **Secure Boot**, **measured boot with a TPM**, and **disk encryption** protect the boot chain and the data at rest. This page explains how they work and how to check them on Linux. For the boot process itself see [BIOS, UEFI, and the Boot Sequence](../../part-2-system-administration/06-boot-grub-systemd/01-bios-uefi-and-boot-sequence.md).

## Key concepts

### UEFI Secure Boot

With Secure Boot, the firmware **only runs signed code** whose signature chains to a trusted key.

```
UEFI firmware (keys: PK, KEK, db, dbx)
   │ verifies signature
   ▼
shim (signed by the Microsoft UEFI CA)  ── holds the distribution's key and the MOK list
   │ verifies
   ▼
GRUB (signed by the distribution)
   │ verifies
   ▼
Linux kernel (signed by the distribution)
   │ verifies (with kernel lockdown)
   ▼
Kernel modules (signed)
```

| Component | Role |
|-----------|------|
| **PK** (Platform Key) | Owner of the platform. Controls who may change the key lists |
| **KEK** | Keys allowed to update `db` and `dbx` |
| **db** | Allowed signatures and hashes |
| **dbx** | **Revoked** signatures and hashes (known-bad bootloaders) |
| **shim** | A small first-stage loader signed with a key trusted by firmware on most hardware. It lets Linux distributions boot without each needing their key in firmware |
| **MOK** (Machine Owner Key) | Keys that **you** enroll to trust (for custom kernels or third-party drivers) |
| **SBAT** | Revocation metadata that lets vendors block vulnerable bootloader generations |

### Kernel lockdown

When Secure Boot is active, the kernel normally enters **lockdown mode**, which prevents even root from changing the running kernel:

| Mode | Effect |
|------|--------|
| `none` | No restrictions |
| `integrity` | Blocks unsigned module loading, raw memory access (`/dev/mem`), `kexec` of unsigned kernels, hibernation images, some debugging interfaces |
| `confidentiality` | Also blocks interfaces that could leak kernel memory (more tracing and debugging features) |

```console
$ cat /sys/kernel/security/lockdown
none [integrity] confidentiality
```

### Measured boot and the TPM

A **TPM** (Trusted Platform Module) is a small secure chip. During boot, firmware and bootloader **measure** (hash) each component into **PCRs** (Platform Configuration Registers). The values cannot be reset without rebooting, and they can be used to:

- **Prove** the boot state to a remote system (**remote attestation**, for example with Keylime).
- **Seal secrets** (such as disk encryption keys) so they are released **only** when the boot chain is unchanged.

| PCR | Typical content |
|-----|-----------------|
| 0, 2 | Firmware code |
| 1, 3 | Firmware configuration |
| 4 | Bootloader |
| 5 | Boot loader configuration and GPT |
| 7 | **Secure Boot state and keys** (common choice for disk unlock) |
| 8–9 | Kernel command line and initrd (GRUB) |
| 11 | Unified kernel image (systemd-stub) |

### Full-disk encryption (LUKS)

**LUKS** (Linux Unified Key Setup, via `dm-crypt`) encrypts block devices. It protects **data at rest**: a stolen disk or laptop is unreadable. It does not protect a running system.

| Concept | Meaning |
|---------|---------|
| **Master key** | Encrypts the data. Stored encrypted in the LUKS header |
| **Key slots** | Up to 8 passphrases or key files that can unlock the master key |
| **Header** | Contains the key slots and parameters. **Back it up**: damage means total data loss |
| `/etc/crypttab` | Lists encrypted devices to unlock at boot |

### Unified Kernel Images and systemd-boot

A **Unified Kernel Image (UKI)** bundles the kernel, initramfs, and command line into **one signed file**, which closes the gap that the initramfs and command line are otherwise unsigned. It is used with `systemd-boot` or directly from UEFI. Distributions are moving in this direction.

## Commands

### Check Secure Boot, lockdown, and TPM

```console
$ mokutil --sb-state
SecureBoot enabled
$ bootctl status 2>/dev/null | head -20              # firmware, Secure Boot, TPM, loader information
$ dmesg | grep -i -E 'secure boot|lockdown|tpm'
$ cat /sys/kernel/security/lockdown
$ ls /sys/class/tpm/                                 # tpm0 present?
$ cat /sys/class/tpm/tpm0/tpm_version_major          # 2 = TPM 2.0
$ tpm2_pcrread sha256:0,4,7                          # PCR values (package tpm2-tools)
$ efibootmgr -v                                      # boot entries (UEFI)
```

### Machine Owner Keys and module signing

If you use third-party kernel modules (for example proprietary GPU drivers or DKMS modules) with Secure Boot on, you must sign them and enroll the key:

```console
# mokutil --list-enrolled                            # keys already trusted
# mokutil --import /var/lib/dkms/mok.pub             # request enrollment (asks for a one-time password)
# reboot                                             # in the blue MOK Manager screen: Enroll MOK, enter the password
$ modinfo mymodule | grep -E 'signer|sig_key|sig_hashalgo'
$ dmesg | grep -i 'module verification failed'       # unsigned module rejected or tainted
```

Many distributions create the DKMS signing key automatically on first use.

### Check revocations

```console
# mokutil --dbx                                      # revoked entries
# fwupdmgr get-devices | grep -i -A3 dbx             # firmware updates include dbx updates
# fwupdmgr update
```

### Encrypt a data disk with LUKS

> [!WARNING]
> `luksFormat` **destroys all data** on the target. Verify the device with `lsblk -f`.

```console
# cryptsetup luksFormat /dev/sdb1                    # asks for a passphrase (type YES in capital letters)
# cryptsetup luksOpen /dev/sdb1 secure               # creates /dev/mapper/secure
# mkfs.ext4 /dev/mapper/secure
# mount /dev/mapper/secure /mnt/secure
# umount /mnt/secure && cryptsetup luksClose secure
```

Inspect and manage:

```console
# cryptsetup luksDump /dev/sdb1                      # header, key slots, cipher (aes-xts-plain64, argon2id)
# cryptsetup luksAddKey /dev/sdb1                    # add another passphrase
# cryptsetup luksRemoveKey /dev/sdb1                 # remove one
# cryptsetup luksHeaderBackup /dev/sdb1 --header-backup-file /safe/place/sdb1.header
# cryptsetup luksHeaderRestore /dev/sdb1 --header-backup-file /safe/place/sdb1.header
# cryptsetup status secure
```

Unlock at boot with `/etc/crypttab`, then mount via `/etc/fstab`:

```
# /etc/crypttab
secure  UUID=2b1c3e9a-...  none  luks,discard
```

Use `blkid` for the UUID. Rebuild the initramfs if the root volume is encrypted (`update-initramfs -u` or `dracut -f`).

### Unlock automatically with the TPM (systemd-cryptenroll)

```console
# systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/nvme0n1p3     # bind the key to Secure Boot state
# systemd-cryptenroll /dev/nvme0n1p3                                      # list enrolled methods
# systemd-cryptenroll --recovery-key /dev/nvme0n1p3                       # create a recovery key (store safely)
```

Then set `tpm2-device=auto` in the `crypttab` options. The disk unlocks without a passphrase only if the measured boot chain is unchanged. **A recovery key or passphrase must exist.** Firmware updates and Secure Boot key changes can alter the PCR values.

### Other boot protections

| Measure | How |
|---------|-----|
| **Firmware (UEFI/BIOS) password** | Stops changes to boot order and Secure Boot settings |
| **GRUB password** | Prevents editing boot parameters at the console (see [GRUB](../../part-2-system-administration/06-boot-grub-systemd/02-grub-and-kernel-parameters.md)) |
| **Disable booting from USB/network** | In firmware, when not needed |
| **Signed UKIs** | Cover the kernel, initramfs, and command line in one signature |
| **IMA/EVM** | Integrity measurement of files at run time (advanced) |
| **Physical protection** | Locked racks, tamper-evident seals, console access control |

## What Secure Boot does and does not do

| Protects against | Does **not** protect against |
|------------------|-------------------------------|
| Bootkits and unsigned bootloaders | Exploits in a running, signed kernel or application |
| Unsigned kernel modules (with lockdown) | Malware running as root after boot |
| Tampering with the kernel on disk | Weak passwords, misconfiguration |
| | An unencrypted disk read offline (needs LUKS) |
| | Tampering with the unsigned **initramfs** and kernel command line on non-UKI systems (partially mitigated by measured boot) |

## Common pitfalls

- **Disabling Secure Boot** because a driver "does not load" instead of signing the module.
- **Losing LUKS passphrases and the header backup.** There is **no recovery** for lost keys. Keep recovery keys and header backups in a safe place.
- **TPM-bound unlock without a fallback** breaks after firmware or Secure Boot changes.
- **Enrolling a MOK key without protecting it.** Anyone who has the signing key can sign modules the system will trust.
- **Cloud and virtual machines.** Many platforms offer virtual Secure Boot and vTPM ("shielded VMs", "Trusted Launch"). Enable them where supported.
- **Assuming full-disk encryption protects a running server.** If the system is on and unlocked, data is accessible to anyone with access to the OS.
- **Encrypting servers that must reboot unattended** without a plan (network-bound unlock with Clevis/Tang, TPM, or remote console entry).
- **Dual boot with revoked bootloaders** (dbx updates can stop old installation media from booting).
- **Ignoring firmware updates.** Vulnerabilities in UEFI firmware and bootloaders are patched through `fwupd` and vendor updates.

## Further reading

- [UEFI Secure Boot (kernel docs)](https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html)
- [`mokutil(1)`](https://man7.org/linux/man-pages/man1/mokutil.1.html)
- [`cryptsetup(8)`](https://man7.org/linux/man-pages/man8/cryptsetup.8.html)
- [`systemd-cryptenroll(1)`](https://man7.org/linux/man-pages/man1/systemd-cryptenroll.1.html)
- [Linux kernel lockdown](https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html)
- [Linux TPM / `tpm2-tools`](https://tpm2-tools.readthedocs.io)
- [Clevis and Tang (network-bound disk encryption)](https://github.com/latchset/clevis)
