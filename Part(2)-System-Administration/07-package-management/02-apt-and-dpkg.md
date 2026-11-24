# APT and dpkg (Debian-based)

## Overview

On Debian, Ubuntu, and derivatives, **APT** is the high-level package manager, and **dpkg** is the low-level tool. All installing and removing commands **require root**.

> [!NOTE]
> Use `apt` for interactive work. Use `apt-get` and `apt-cache` in scripts because their output format and options are more stable.

## Commands

### The daily cycle

```console
# apt update                  # download the newest package lists
# apt upgrade                 # upgrade installed packages (never removes packages)
# apt full-upgrade            # upgrade, and add or remove packages if needed (like dist-upgrade)
# apt install nginx           # install
# apt install nginx=1.24.0-2  # a specific version
# apt remove nginx            # remove program, keep configuration
# apt purge nginx             # remove program and configuration
# apt autoremove              # remove automatically installed packages nobody needs now
# apt clean                   # delete downloaded .deb files
```

```console
# apt update
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Get:2 http://archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB]
Fetched 1,043 kB in 1s (1,020 kB/s)
Reading package lists... Done
12 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

> [!IMPORTANT]
> `apt update` only refreshes the **lists**. It does not upgrade anything. Always run it before `apt install` or `apt upgrade`.

### Search and information

| Command | Purpose |
|---------|---------|
| `apt search WORD` | Search names and descriptions |
| `apt show PKG` | Details: version, dependencies, size, description |
| `apt list --installed` | Installed packages |
| `apt list --upgradable` | Packages with updates |
| `apt policy PKG` | Installed and candidate versions and their repositories |
| `apt-cache depends PKG` | Dependencies |
| `apt-cache rdepends PKG` | What depends on it |
| `apt-file search /usr/bin/dig` | Find which package contains a file (install `apt-file`, run `apt-file update`) |
| `apt-mark showmanual` | Packages you installed by hand |

```console
$ apt policy nginx
nginx:
  Installed: 1.24.0-2ubuntu7
  Candidate: 1.24.0-2ubuntu7.1
  Version table:
     1.24.0-2ubuntu7.1 500
        500 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
 *** 1.24.0-2ubuntu7 500
        500 http://archive.ubuntu.com/ubuntu noble/main amd64 Packages
```

### dpkg: queries and single files

| Command | Purpose |
|---------|---------|
| `dpkg -l` | List installed packages |
| `dpkg -l 'nginx*'` | List matching packages |
| `dpkg -L PKG` | Files installed by a package |
| `dpkg -S /usr/sbin/nginx` | Which package owns a file |
| `dpkg -s PKG` | Status and details |
| `dpkg -i FILE.deb` | Install a local file (no dependency resolution) |
| `dpkg -r PKG` / `dpkg -P PKG` | Remove / purge |
| `dpkg --configure -a` | Finish interrupted installs |
| `dpkg-reconfigure PKG` | Run the package's configuration questions again |

Install a local `.deb` **with** dependency handling:

```console
# apt install ./google-chrome-stable_current_amd64.deb
```

### Hold a package at its version

```console
# apt-mark hold nginx
# apt-mark showhold
# apt-mark unhold nginx
```

### Upgrading to a new release

```console
# do-release-upgrade             # Ubuntu, from one release to the next
# apt full-upgrade               # Debian: after editing sources to the new release
```

Read the release notes first. Take a backup or snapshot.

## Configuration files

| Path | Purpose |
|------|---------|
| `/etc/apt/sources.list` | Classic one-line repository list |
| `/etc/apt/sources.list.d/*.list` | More one-line sources |
| `/etc/apt/sources.list.d/*.sources` | **deb822** format (default on recent Ubuntu and Debian) |
| `/etc/apt/preferences.d/` | Pinning (priorities) |
| `/etc/apt/apt.conf.d/` | APT options (proxy, unattended upgrades) |
| `/etc/apt/keyrings/`, `/usr/share/keyrings/` | Repository signing keys |
| `/var/lib/dpkg/status` | The installed-package database |

Proxy example (`/etc/apt/apt.conf.d/90proxy`):

```
Acquire::http::Proxy "http://proxy.example.com:3128";
```

## Common pitfalls

- **"Could not get lock /var/lib/dpkg/lock-frontend".** Another process (often an automatic update) is running. Wait, or find it with `ps aux | grep -E 'apt|dpkg'`. Do **not** delete the lock file unless you are sure nothing is running.
- **Interrupted upgrade.** Run `dpkg --configure -a` and then `apt -f install`.
- **"Unable to locate package".** Run `apt update`. Check the exact name with `apt search`. The repository (for example `universe`) may be disabled.
- **"held broken packages".** A dependency conflict, often from a third-party repository. Read the message and use `apt policy` to inspect versions.
- **`apt upgrade` keeps back packages.** They need new dependencies. Review them, then use `apt full-upgrade`.
- **Old kernels fill `/boot`.** `apt autoremove --purge` removes unneeded kernels. Keep at least one previous kernel.
- **Prompts in scripts.** Use `apt-get -y install PKG` with `DEBIAN_FRONTEND=noninteractive`.
- **`apt-key` is deprecated.** Store keys as files and reference them with `signed-by=` (see [Repositories](05-repositories.md)).

## Further reading

- [`apt(8)`](https://manpages.debian.org/stable/apt/apt.8.en.html)
- [`dpkg(1)`](https://man7.org/linux/man-pages/man1/dpkg.1.html)
- [Debian Wiki: SourcesList](https://wiki.debian.org/SourcesList)
- [Ubuntu Server: package management](https://documentation.ubuntu.com/server/how-to/software/package-management/)
