# Distribution Release Cycles and Support Lifecycles

## Overview

Every distribution release has a **lifetime**. When support ends, you no longer get security fixes. Knowing the release model helps you plan upgrades and avoid running unsupported systems.

## Key concepts

### Release models

| Model | How it works | Good for | Examples |
|-------|--------------|----------|----------|
| **Fixed release** | A new version is published on a schedule. Packages stay mostly the same within a release. | Servers needing stability | Debian stable, Ubuntu, RHEL |
| **Long-term support (LTS)** | A fixed release supported for many years | Production systems | Ubuntu LTS, RHEL, SLES |
| **Rolling release** | No version numbers. Packages update continuously. | Desktops, developers | Arch Linux, openSUSE Tumbleweed |
| **Fast release** | Short support window, new features quickly | Testing, early access | Fedora |

### Typical lifecycles

> [!IMPORTANT]
> Support periods change. The figures below are the usual patterns. **Always check the official lifecycle page** before you plan.

| Distribution | Release pattern | Support pattern |
|--------------|----------------|-----------------|
| Debian | New stable release about every two years | Full support for about 3 years, then Long Term Support (LTS) from volunteers, about 5 years in total |
| Ubuntu | A release every 6 months; **LTS every 2 years** (April of even years) | LTS: 5 years of standard security maintenance, extendable with Ubuntu Pro |
| RHEL | A major release about every 3 years; minor releases about every 6 months | About 10 years for a major release, plus paid extensions |
| Rocky / AlmaLinux | Follow RHEL | Follow RHEL's major-release lifetime |
| Fedora | Every 6 months or so | About 13 months per release |
| SLES | Major release with service packs or minor versions | Long support, with paid extensions |
| openSUSE Leap | Follows SLES base | Shorter than SLES |

### Terms to know

- **EOL (End of Life):** No more updates of any kind.
- **Standard support / maintenance:** Security fixes and bug fixes.
- **Extended support:** Paid or limited fixes after standard support ends.
- **Point release / minor release:** An update within a major version (for example, RHEL 9.4 → 9.5).
- **Backporting:** Applying a security fix from a newer software version to the older version shipped in a release, so the release stays stable.
- **In-place upgrade:** Upgrading from one major version to the next on the same system.

## Commands

Check the version you are running:

```console
$ cat /etc/os-release | grep -E 'VERSION_ID|PRETTY_NAME'
PRETTY_NAME="Rocky Linux 9.4 (Blue Onyx)"
VERSION_ID="9.4"
```

Ubuntu release and support information:

```console
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04 LTS
Release:        24.04
Codename:       noble
```

> [!NOTE]
> `lsb_release` may need the `lsb-release` package on some systems. `/etc/os-release` works everywhere.

## Common pitfalls

- **Running an EOL system in production.** No security fixes means growing risk. Plan upgrades **before** the EOL date.
- **Choosing a short-life release for a long-lived server.** Fedora and non-LTS Ubuntu releases are not meant for servers that must run for years.
- **Judging security only by version number.** Because of backporting, an "old" package version may already include the fix. Check the vendor's security tracker.

## Further reading

- [Ubuntu release cycle](https://ubuntu.com/about/release-cycle)
- [Debian releases](https://www.debian.org/releases/)
- [Red Hat Enterprise Linux life cycle](https://access.redhat.com/support/policy/updates/errata)
- [Fedora releases](https://docs.fedoraproject.org/en-US/releases/)
- [endoflife.date](https://endoflife.date) (community-maintained lifecycle dates)
