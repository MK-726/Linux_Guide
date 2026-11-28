# SELinux: Policies, Booleans, and Customization

## Overview

The SELinux **policy** is a large set of rules shipped by the distribution. You rarely write policy. Most customization is done with four simple tools: **booleans** (switches for common options), **file contexts** (labels by path), **port labels**, and, only when needed, **custom policy modules**. This page shows each and when to use it.

## Key concepts

### The order of fixes (from simplest to most invasive)

1. **Wrong label on files or directories** → fix with `semanage fcontext` + `restorecon`.
2. **Service needs a feature the policy allows optionally** → turn on a **boolean**.
3. **Service uses a non-standard port** → label the **port** with `semanage port`.
4. **Service needs something the policy does not allow at all** → use a different supported layout, or a **custom module** (carefully).
5. **Last resort for one service:** a **permissive domain** (`semanage permissive -a TYPE_t`) while you develop policy. Never leave it permanent without a reason.

Most problems end at step 1, 2, or 3.

### Policy parts

| Term | Meaning |
|------|---------|
| **Domain** | The type of a **process** (for example `httpd_t`) |
| **Type enforcement rule** | `allow SOURCE_TYPE TARGET_TYPE:CLASS { PERMISSIONS };` |
| **Class** | Kind of object: `file`, `dir`, `tcp_socket`, `process`, ... |
| **Boolean** | A switch in the policy that turns groups of rules on or off |
| **Module** | A package of policy rules (base policy plus modules for each service) |
| **Transition** | A rule that changes the domain when a program runs (`init_t` runs `/usr/sbin/httpd` → `httpd_t`) |
| **Interface** | A reusable macro used when writing policy |

## Commands

### Booleans

```console
$ getsebool -a | grep httpd | head
httpd_can_network_connect --> off
httpd_can_network_connect_db --> off
httpd_can_sendmail --> off
httpd_enable_homedirs --> off
$ getsebool httpd_can_network_connect
# setsebool httpd_can_network_connect on              # runtime only
# setsebool -P httpd_can_network_connect on           # -P: permanent (survives reboot)
# semanage boolean -l | grep httpd_can_network        # with descriptions and the current/default values
httpd_can_network_connect  (off , off)  Allow HTTPD scripts and modules to connect to the network using TCP.
# semanage boolean -l -C                              # only booleans you changed from the default
```

Common booleans:

| Boolean | Purpose |
|---------|---------|
| `httpd_can_network_connect` | Web app may connect to other hosts (APIs, proxy upstreams) |
| `httpd_can_network_connect_db` | Web app may connect to database ports |
| `httpd_can_sendmail` | Web app may send mail |
| `httpd_enable_homedirs` | Serve content from user home directories |
| `httpd_unified` | Treat all httpd types as one (loosens separation) |
| `httpd_use_nfs`, `httpd_use_cifs` | Web content on NFS or SMB |
| `nfs_export_all_rw`, `use_nfs_home_dirs` | NFS server exports / NFS home directories |
| `samba_enable_home_dirs`, `samba_export_all_rw` | Samba and home directories / read-write shares |
| `ftpd_full_access`, `ftp_home_dir` | FTP server access |
| `container_manage_cgroup` | Containers manage cgroups (some systemd-in-container cases) |
| `virt_use_nfs`, `virt_use_samba` | Virtual machines with network storage |
| `ssh_sysadm_login` | Administrators log in with the `sysadm_r` role |
| `allow_user_exec_content` | Confined users may execute files in their home |

Only enable what you need. Use `man SERVICE_selinux` to read the booleans and types for each service.

### File contexts (labels by path)

```console
# semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"            # read-only web content
# semanage fcontext -a -t httpd_sys_rw_content_t "/srv/web/uploads(/.*)?" # content the server may write
# semanage fcontext -a -t mysqld_db_t "/data/mysql(/.*)?"                 # database files in a custom location
# semanage fcontext -a -t var_log_t "/opt/app/logs(/.*)?"
# restorecon -Rv /srv/web /data/mysql
# semanage fcontext -l -C                                                  # your custom rules
# semanage fcontext -d "/srv/web(/.*)?"                                    # delete a rule
# matchpathcon /srv/web/uploads/a.txt
# restorecon -Rvn /srv/web                                                 # -n: dry run (show changes only)
```

The pattern `"/path(/.*)?"` is a regular expression that matches the directory and everything under it.

Equivalence rules: make one tree labeled **like another** (for example, when moving a data directory):

```console
# semanage fcontext -a -e /var/lib/mysql /data/mysql        # label /data/mysql as /var/lib/mysql
# restorecon -Rv /data/mysql
```

### Port labels

```console
# semanage port -l | grep -w http_port_t
http_port_t   tcp  80, 81, 443, 488, 8008, 8009, 8443, 9000
# semanage port -a -t http_port_t -p tcp 8888              # allow httpd_t to bind 8888
# semanage port -a -t ssh_port_t -p tcp 2222               # SSH on 2222
# semanage port -m -t http_port_t -p tcp 9000              # modify a port that already has another label
# semanage port -d -t http_port_t -p tcp 8888
# semanage port -l -C                                      # your changes
```

If the port is already assigned to another type, use `-m` (modify), not `-a`.

### Custom policy modules (when a boolean and labels are not enough)

Only after you understand the denial. Generate a module from **one specific** denial:

```console
# ausearch -m AVC -ts recent | audit2allow -w          # explain the denial in plain words
# ausearch -m AVC -ts recent | audit2allow -R          # show suggested rules
# ausearch -m AVC -c 'myapp' --raw | audit2allow -M myapp_local     # create myapp_local.te and myapp_local.pp
# cat myapp_local.te                                    # READ and review the rules before installing!
module myapp_local 1.0;
require {
        type myapp_t;
        type var_lib_t;
        class file { read open getattr };
}
allow myapp_t var_lib_t:file { read open getattr };
# semodule -i myapp_local.pp                            # install
# semodule -l | grep myapp_local
# semodule -r myapp_local                               # remove
```

> [!WARNING]
> **Never pipe all denials into `audit2allow` and install the result without review.** It will happily allow dangerous accesses that an attacker used or that point to a real bug (a mislabeled file would be fixed with a label, not a rule). Prefer labels, booleans, and ports over custom modules.

For an application you control, write a proper policy using `sepolicy generate` as a starting point:

```console
# sepolicy generate --init /opt/myapp/bin/myapp          # creates a template (myapp.te, myapp.if, myapp.fc, myapp.sh)
```

### Inspecting the policy

```console
# sesearch -A -s httpd_t -t httpd_sys_content_t -c file       # which rules allow httpd_t to access that type?
# sesearch -A -s httpd_t -b httpd_can_network_connect         # rules enabled by a boolean
# seinfo -t | grep -c .                                       # number of types
# sepolicy network -t http_port_t                             # which domains may use this port type
# sepolicy transition -s init_t -t httpd_t                    # how a domain transition happens
# semodule -l | head                                          # installed modules
# semodule -DB                                                # disable "dontaudit" rules (see hidden denials); restore with: semodule -B
```

### Containers and volumes

For Podman/Docker volumes, label host directories with the container file type through the mount option:

```console
$ podman run -v /srv/data:/data:Z myimage          # Z: private label for this container
$ podman run -v /srv/shared:/data:z myimage        # z: shared label for several containers
```

`:Z` relabels the host directory. **Never use it on system directories** such as `/home` or `/usr`. Use a dedicated data directory.

### Policy for systemd services started from non-standard paths

Executables in `/opt` or `/usr/local` often run in the wrong domain (unconfined or a generic type) or are denied. Label them to match a standard binary type:

```console
# semanage fcontext -a -t bin_t "/opt/myapp/bin(/.*)?"
# restorecon -Rv /opt/myapp/bin
```

If the service should run in its own domain, a custom policy (`sepolicy generate`) is required.

## Common pitfalls

- **Turning on `httpd_unified` or too many booleans** to make things work. Loosening removes protection.
- **Using `audit2allow` blindly.** Review each rule. Look first for a label, boolean, or port fix.
- **Forgetting `-P`** on `setsebool`. The change disappears at reboot.
- **Forgetting `restorecon`** after `semanage fcontext`. The rule exists but files keep old labels.
- **`semanage port -a` for a port that is already defined** returns an error. Use `-m`.
- **Mislabeling with `chcon -R` on large trees.** Use `semanage fcontext` so the labels survive relabeling.
- **Policy modules leftover** from old troubleshooting (`semodule -l`). Remove modules you do not need.
- **`dontaudit` rules hide denials.** If something fails with no AVC, run `semodule -DB`, reproduce, then `semodule -B`.
- **Policy differences between releases.** After upgrades, booleans and types can change. Re-test.
- **Applying `:Z` to a shared or system directory** relabels it for one container and breaks others.

## Further reading

- [`semanage-boolean(8)`](https://man7.org/linux/man-pages/man8/semanage-boolean.8.html)
- [`semanage-fcontext(8)`](https://man7.org/linux/man-pages/man8/semanage-fcontext.8.html)
- [`audit2allow(1)`](https://man7.org/linux/man-pages/man1/audit2allow.1.html)
- [Red Hat: customizing SELinux policy](https://docs.redhat.com)
- [SELinux Notebook: policy language](https://github.com/SELinuxProject/selinux-notebook)
