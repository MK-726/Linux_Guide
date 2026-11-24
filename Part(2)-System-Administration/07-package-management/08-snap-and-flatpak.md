# Snap and Flatpak

## Overview

**Snap** and **Flatpak** are "universal" package formats. An application is packaged **with its own libraries** and runs in a **sandbox**. The same package works on many distributions. They are most useful for desktop applications and for software that is not in your distribution's repository. They are less common on servers.

## Key concepts

| | **Snap** | **Flatpak** |
|---|----------|-------------|
| Developer | Canonical | Community (freedesktop.org, Red Hat, GNOME) |
| Store | Snap Store | Flathub and others |
| Default on | Ubuntu | Fedora Workstation (Flatpak available), many desktop distributions |
| Server and CLI software | Yes (common for tools and services) | Mostly desktop apps |
| Updates | **Automatic** (several times a day, by default) | Manual or desktop-integrated |
| Sandbox | AppArmor, seccomp, cgroups | bubblewrap, portals, seccomp |

### Why use them

- Newer application versions than the distribution offers.
- Same package across distributions.
- Isolation from the rest of the system.

### Trade-offs

- Larger disk use (each app bundles its libraries).
- Slower first start.
- Sandboxing can block file or device access.
- Snap auto-updates can surprise administrators.
- Snap appears as many loop devices in `df` and `lsblk`.

## Commands

### Snap

Install support (Ubuntu has it already):

**Debian-based:** `apt install snapd`
**RHEL-based:** enable EPEL, then `dnf install snapd` and `systemctl enable --now snapd.socket`

| Command | Purpose |
|---------|---------|
| `snap find WORD` | Search the store |
| `snap info NAME` | Details and available channels |
| `snap install NAME` | Install (**root**) |
| `snap install NAME --classic` | Install with less confinement (needs approval by the publisher) |
| `snap install NAME --channel=1.28/stable` | Choose a channel |
| `snap list` | Installed snaps |
| `snap refresh` | Update all snaps |
| `snap refresh --hold=72h` | Delay updates (also `snap refresh --hold`) |
| `snap remove NAME` | Remove |
| `snap services` | Services provided by snaps |
| `snap logs NAME` | Service logs |
| `snap connections NAME` | Permissions (interfaces) of a snap |

```console
$ snap list
Name    Version   Rev   Tracking       Publisher   Notes
core22  20240111  1122  latest/stable  canonical   base
lxd     5.21.1    29351 5.21/stable    canonical   -

# snap install --classic kubectl
```

| Confinement | Meaning |
|-------------|---------|
| **strict** | Fully sandboxed; access only through declared interfaces |
| **classic** | Normal system access (no sandbox), for developer tools |
| **devmode** | For development; not for production |

Control update timing (**root**):

```console
# snap set system refresh.timer=sat,03:00-05:00        # update only Saturday 03:00-05:00
# snap refresh --time                                   # show the schedule
```

### Flatpak

```console
# apt install flatpak                    # Debian-based
# dnf install flatpak                    # RHEL-based
# flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

| Command | Purpose |
|---------|---------|
| `flatpak search WORD` | Search |
| `flatpak install flathub org.example.App` | Install |
| `flatpak install --user flathub org.example.App` | Install for one user only (no root) |
| `flatpak run org.example.App` | Run |
| `flatpak list` | Installed applications and runtimes |
| `flatpak update` | Update |
| `flatpak uninstall org.example.App` | Remove |
| `flatpak uninstall --unused` | Remove runtimes no longer used |
| `flatpak info org.example.App` | Details |
| `flatpak override --user --filesystem=/data org.example.App` | Change sandbox permissions |
| `flatpak permissions` | Show granted permissions |

Application IDs look like `org.mozilla.firefox`. Apps usually install under `/var/lib/flatpak` (system) or `~/.local/share/flatpak` (user).

### Disk usage

```console
$ du -sh /var/lib/snapd /var/lib/flatpak ~/.local/share/flatpak 2>/dev/null
$ df -h | grep -v loop            # hide Snap loop mounts
$ lsblk -e7                        # hide loop devices (type 7)
```

## Common pitfalls

- **Snap loop devices clutter `df` and `lsblk`.** Filter them out with `grep -v loop` or `lsblk -e7`.
- **Unexpected Snap updates on servers.** Set a refresh window or hold updates, and test in staging.
- **A sandboxed app cannot read your files.** Check `snap connections NAME` or `flatpak info --show-permissions`, then grant the needed access.
- **Mixing a Snap and a package of the same program** causes confusion over which version runs. Check `type -a program`.
- **Old revisions use disk space.** Snap keeps old revisions. Lower the number: `snap set system refresh.retain=2`.
- **Security review.** Check the publisher. Some Flathub and Snap Store apps are community-maintained, not from the original project.
- **Not all environments allow them.** Minimal servers, containers, and some hardened systems may not.

## Further reading

- [Snapcraft documentation](https://snapcraft.io/docs)
- [Flatpak documentation](https://docs.flatpak.org)
- [Flathub](https://flathub.org)
