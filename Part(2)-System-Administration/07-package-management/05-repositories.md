# Repositories and Repository Configuration

## Overview

Repositories are the source of your software. Who you trust, how they are signed, and which one wins when versions conflict decide how safe and stable your system is. Review every third-party repository before you add it.

## Key concepts

| Term | Meaning |
|------|---------|
| **Official repositories** | Provided by the distribution and signed with its key |
| **Third-party repositories** | Provided by software vendors (Docker, PostgreSQL, Nginx, Microsoft, and others) |
| **Community repositories** | EPEL, PPA (Ubuntu), openSUSE Build Service (OBS), Packman |
| **Mirrors** | Copies of a repository on other servers (choose a near one) |
| **Local/internal repositories** | Your own mirror (Satellite, Pulp, aptly, Nexus, Artifactory) for control and offline use |
| **Release channels** | For example `main`, `updates`, `security`, `backports` |

### How trust works

1. The repository publishes a **signed index** (APT: `Release`/`InRelease`; DNF: `repomd.xml`).
2. Your system has the **public key** of the repository.
3. The package manager verifies the signature before it uses the index and the packages. A key you add gives that repository the power to install **any** software as root on your system.

> [!WARNING]
> Adding a repository and its key is a **security decision**. Install only from sources you trust, and prefer pinning the key to that one repository.

## Commands and configuration

### APT (Debian-based)

**deb822 format** (`/etc/apt/sources.list.d/*.sources`), used by recent Ubuntu and Debian:

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

**One-line format** (`/etc/apt/sources.list.d/docker.list`):

```
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable
```

Add a key and repository (example pattern):

```console
# install -m 0755 -d /etc/apt/keyrings
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
# chmod a+r /etc/apt/keyrings/docker.asc
# apt update
```

Always follow the **vendor's current installation page** for the exact URLs.

| Item | Meaning |
|------|---------|
| `Suites` / suite name | Release codename and pocket: `noble`, `noble-updates`, `noble-security` |
| `Components` | `main`, `universe`, `restricted`, `multiverse` (Ubuntu); `main`, `contrib`, `non-free`, `non-free-firmware` (Debian) |
| `signed-by` | Only this key is trusted for this repository |

Ubuntu PPAs:

```console
# add-apt-repository ppa:ondrej/php
# add-apt-repository --remove ppa:ondrej/php
```

PPAs are personal repositories with no official review. Use them sparingly on servers.

**Pinning (priorities)** in `/etc/apt/preferences.d/`:

```
Package: *
Pin: origin "download.docker.com"
Pin-Priority: 100
```

Higher number wins (default 500; 1000 allows downgrade). Check the result with `apt policy`.

### DNF (RHEL-based)

`/etc/yum.repos.d/example.repo`:

```ini
[example]
name=Example Repository
baseurl=https://repo.example.com/el9/$basearch/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://repo.example.com/RPM-GPG-KEY-example
priority=50
```

| Option | Meaning |
|--------|---------|
| `[id]` | Unique repository ID |
| `baseurl` or `mirrorlist` or `metalink` | Where packages are |
| `enabled` | 1 = on, 0 = off |
| `gpgcheck` | Verify package signatures (**keep at 1**) |
| `gpgkey` | Key location |
| `priority` | Lower number = higher priority (needs the priority feature, built in for DNF) |
| `exclude=` / `includepkgs=` | Limit what comes from this repository |

Variables: `$releasever` (release), `$basearch` (architecture).

```console
# dnf repolist --all
# dnf config-manager --set-enabled crb
# dnf config-manager --set-disabled example
# dnf --disablerepo='*' --enablerepo=example install PKG     # use only one repo for this command
# rpm --import https://repo.example.com/RPM-GPG-KEY-example
# dnf makecache
```

`dnf config-manager` comes from `dnf-plugins-core` (DNF 4). DNF5 uses slightly different syntax for adding repositories, so check `dnf config-manager --help`.

### Zypper (SUSE-based)

```console
# zypper addrepo -f -p 90 https://download.example.com/opensuse/ example
# zypper refresh
# zypper repos -P            # list with priorities
```

`-f` means auto-refresh. A lower priority number wins (default 99).

### Create your own repository (basic idea)

| Family | Tool |
|--------|------|
| RHEL-based / SUSE-based | `createrepo_c /srv/repo/` creates `repodata/` |
| Debian-based | `aptly`, `reprepro`, or `dpkg-scanpackages` for simple cases |

Serve the directory with Nginx or Apache over HTTPS, and sign the repository.

## Common pitfalls

- **Repository for the wrong release.** Using `jammy` entries on `noble` can break upgrades. Match your release codename.
- **Old key methods.** `apt-key add` is deprecated. Use `signed-by`.
- **`gpgcheck=0`.** Disables verification and allows tampered packages. Do not use it for convenience.
- **Third-party repository replaces system libraries.** Use priorities or `includepkgs=` to limit scope.
- **Dead repositories after end of life.** After EOL, mirrors move to archives (for example `vault.centos.org` or `old-releases.ubuntu.com`). Upgrade the system instead of just redirecting sources.
- **`add-apt-repository` and PPAs on Debian.** PPAs are made for Ubuntu and may not work on Debian.
- **Proxy and certificate problems.** Check `curl -I REPO_URL` from the same host.
- **Disabling the security repository** to "fix" a conflict leaves the system unpatched.

## Further reading

- [Debian Wiki: SourcesList](https://wiki.debian.org/SourcesList)
- [`sources.list(5)` (deb822)](https://manpages.debian.org/stable/apt/sources.list.5.en.html)
- [DNF configuration reference](https://dnf.readthedocs.io/en/latest/conf_ref.html)
- [`createrepo_c`](https://github.com/rpm-software-management/createrepo_c)
