# Installing Software Outside Package Managers and Building from Source

## Overview

Sometimes the software you need is not in a repository, or you need a newer version. You can install a vendor archive, build from source, or use a language-specific tool. These installs are **not tracked** by the package manager, so you must manage them yourself. Prefer packages when they exist.

## Key concepts

### Where to put manually installed software

| Location | Use |
|----------|-----|
| `/usr/local/` (`bin`, `lib`, `share`, ...) | Software you build and install system-wide. The package manager never touches it |
| `/opt/<name>/` | Self-contained vendor software with its own directory tree |
| `~/bin`, `~/.local/` | Per-user installs, no root needed |

Never install manual software into `/usr/bin` or `/usr/lib`. Those belong to the package manager.

### Order of preference

1. Official package from the distribution.
2. Official vendor repository (signed).
3. Container image or Snap/Flatpak, where appropriate.
4. Vendor archive in `/opt` (with checksum verification).
5. Build from source into `/usr/local`.

## Commands

### Install a vendor archive

```console
$ curl -fLO https://example.com/tool-2.1.0-linux-amd64.tar.gz
$ curl -fLO https://example.com/tool-2.1.0-linux-amd64.tar.gz.sha256
$ sha256sum -c tool-2.1.0-linux-amd64.tar.gz.sha256
tool-2.1.0-linux-amd64.tar.gz: OK
# mkdir -p /opt/tool-2.1.0
# tar -xzf tool-2.1.0-linux-amd64.tar.gz -C /opt/tool-2.1.0 --strip-components=1
# ln -sfn /opt/tool-2.1.0 /opt/tool                 # stable "current" path
# ln -s /opt/tool/bin/tool /usr/local/bin/tool      # in PATH
```

A symbolic link such as `/opt/tool` makes upgrades and rollbacks easy: extract a new version and move the link.

### Build from source

**1. Install build tools**

| Family | Command |
|--------|---------|
| Debian-based | `apt install build-essential` |
| RHEL-based | `dnf group install "Development Tools"` |
| SUSE-based | `zypper install -t pattern devel_basis` |

Install the **development packages** of libraries the software needs (`-dev` on Debian-based, `-devel` on RHEL-based and SUSE-based). To pull in the same build dependencies as an existing package:

```console
# apt build-dep nginx           # Debian-based (needs deb-src entries)
# dnf builddep nginx.spec       # RHEL-based (dnf-plugins-core)
```

**2. Download, verify, extract**

```console
$ curl -fLO https://example.org/src/myprog-1.4.tar.gz
$ sha256sum myprog-1.4.tar.gz        # compare with the project's published value
$ tar -xzf myprog-1.4.tar.gz
$ cd myprog-1.4
```

**3. Configure, compile, install**

```console
$ less README INSTALL                          # read the project's instructions first
$ ./configure --prefix=/usr/local
$ make -j"$(nproc)"
$ make check                                   # tests, if available
# make install
```

| Step | What it does |
|------|--------------|
| `./configure` | Checks dependencies, sets options (`--prefix`, `--with-ssl`, ...). Run `./configure --help` |
| `make` | Compiles. `-j N` uses N parallel jobs |
| `make install` | Copies files into the prefix |

Other build systems use different commands: `cmake -B build && cmake --build build && cmake --install build`, `meson setup build && ninja -C build install`, `cargo install`, `go install`.

**4. Make libraries findable**

```console
# echo /usr/local/lib > /etc/ld.so.conf.d/local.conf
# ldconfig
$ ldd /usr/local/bin/myprog | grep "not found"      # check for missing libraries
```

### Track and uninstall source installs

`make uninstall` exists in some projects but not all. Better options:

| Option | How |
|--------|-----|
| **GNU Stow** | Install into `/usr/local/stow/myprog-1.4` (`--prefix`), then `stow myprog-1.4` creates symbolic links. `stow -D` removes them |
| **Build a package** | Use `checkinstall`, `fpm`, or write a proper `.deb` / `.rpm` spec |
| **Staging** | `make install DESTDIR=/tmp/stage` lets you inspect what would be installed |

```console
$ make install DESTDIR=/tmp/stage
$ find /tmp/stage -type f
```

### Language-specific package tools

| Language | Tool | Best practice |
|----------|------|---------------|
| Python | `pip`, `pipx` | Use a **virtual environment** (`python3 -m venv`) or `pipx` for command-line tools |
| Node.js | `npm`, `nvm` | Project-local `node_modules` |
| Ruby | `gem`, `bundler` | Bundler per project |
| Go | `go install` | Installs to `~/go/bin` |
| Rust | `cargo install` | Installs to `~/.cargo/bin` |

> [!NOTE]
> Recent distributions mark the system Python as "externally managed". `pip install` outside a virtual environment fails with `externally-managed-environment`. Use `python3 -m venv .venv` or `pipx`. Avoid `--break-system-packages`.

### "curl | bash" installers

```console
$ curl -fsSL https://example.com/install.sh | sudo bash
```

> [!WARNING]
> This runs code from the internet as root without review. Download the script first, read it, verify its checksum if provided, and run it only if you trust the source.

```console
$ curl -fsSLo install.sh https://example.com/install.sh
$ less install.sh
$ sudo bash install.sh
```

## Common pitfalls

- **Overwriting package-managed files.** Install to `/usr/local` or `/opt` so updates do not conflict.
- **No update path.** Manual installs miss security patches. Record version and source in a document, and plan updates.
- **Missing `-dev`/`-devel` packages** cause `configure` errors like "library not found" or "header not found". Install the matching development package.
- **Compiling as root.** Build as a normal user. Use root only for `make install`.
- **Wrong `PATH`.** `/usr/local/bin` should come before `/usr/bin`, or you may run the old version. Check with `type -a program`.
- **Library not found at run time.** Run `ldconfig` and check `/etc/ld.so.conf.d/`.
- **Mixing architectures** (x86_64 and aarch64 archives). Check `uname -m`.
- **Not verifying downloads.** Always check the checksum and, if available, the GPG signature.
- **Forgetting to track installs.** Keep a list so the next administrator knows what was installed manually.

## Further reading

- [GNU Coding Standards: installation directories](https://www.gnu.org/prep/standards/html_node/Directory-Variables.html)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Python packaging: virtual environments](https://packaging.python.org/en/latest/guides/installing-using-pip-and-virtual-environments/)
- [`ldconfig(8)`](https://man7.org/linux/man-pages/man8/ldconfig.8.html)
