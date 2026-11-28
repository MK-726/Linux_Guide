# SELinux: Troubleshooting

## Overview

When SELinux blocks something, the application usually shows a generic "Permission denied" while normal file permissions look correct. The **audit log** shows the real reason. This page gives a repeatable method to find the denial, identify the right fix, and avoid the temptation to disable SELinux.

## Key concepts

### Symptoms that suggest SELinux

- "Permission denied" even though `ls -l`, `id`, and `namei -l` look correct.
- A service fails after you move data, change ports, or install software in non-standard paths.
- It works when you run `setenforce 0` (**this proves SELinux is involved**).
- Log entries `avc: denied` in `/var/log/audit/audit.log` (or the journal).

### Reading an AVC denial

```
type=AVC msg=audit(1741000000.123:456): avc:  denied  { read } for  pid=1234 comm="nginx" name="index.html" dev="sda1" ino=393219 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
```

| Field | Meaning |
|-------|---------|
| `denied { read }` | The **permission** requested and denied |
| `comm="nginx"` | The program |
| `name="index.html"` | The object |
| `scontext=...httpd_t...` | **Source** context: the process domain |
| `tcontext=...user_home_t...` | **Target** context: the file's label |
| `tclass=file` | Class of the target (`file`, `dir`, `tcp_socket`, `process`, ...) |
| `permissive=0` | The system was enforcing. `1` = it would have been denied but was allowed |

Reading it: *"The process `nginx` (type `httpd_t`) tried to read a file labeled `user_home_t` and was denied."* The fix is probably to **relabel** the file as `httpd_sys_content_t`, not to allow `httpd_t` to read home directories.

### Where denials are logged

| Source | Notes |
|--------|-------|
| `/var/log/audit/audit.log` | Primary (needs `auditd` running) |
| Journal (`journalctl _AUDIT_TYPE_NAME=AVC`) | If `auditd` is not running, AVCs go to the kernel log |
| `dmesg` | Fallback |
| `setroubleshoot` (sealert) | Friendly analysis with suggested commands |

## Commands

### The troubleshooting workflow

**1. Confirm that SELinux is the cause (and nothing else changed).**

```console
$ getenforce
# setenforce 0           # TEMPORARILY permissive: if the problem disappears, it is SELinux
# <reproduce the problem>
# setenforce 1           # back to enforcing immediately
```

Better for production: make only the one domain permissive (`semanage permissive -a httpd_t`), then remove it afterwards.

**2. Find the denials.**

```console
# ausearch -m AVC,USER_AVC -ts recent                    # last 10 minutes
# ausearch -m AVC -ts today -c nginx                     # by program
# ausearch -m AVC -ts 10:00:00 -te 10:05:00
# ausearch -m AVC -i -ts recent                          # -i: interpret fields (names)
# journalctl -t setroubleshoot --since "30 min ago"
# grep AVC /var/log/audit/audit.log | tail
```

**3. Understand the denial.**

```console
# ausearch -m AVC -ts recent | audit2why
type=AVC msg=audit(...): avc:  denied  { name_connect } for  pid=1234 comm="php-fpm" dest=5432 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:postgresql_port_t:s0 tclass=tcp_socket
        Was caused by:
        The boolean httpd_can_network_connect_db was set incorrectly.
        Description:
        Allow HTTPD scripts and modules to connect to databases over the network.

        Allow access by executing:
        # setsebool -P httpd_can_network_connect_db 1
```

`audit2why` often names the boolean. With `setroubleshoot-server`, use:

```console
# sealert -a /var/log/audit/audit.log            # readable report with suggestions and confidence levels
# sealert -l '*'                                 # list alerts
```

**4. Apply the correct kind of fix.**

| What the denial shows | Fix |
|-----------------------|-----|
| Target file/dir has the **wrong type** (`user_home_t`, `var_t`, `default_t`, `tmp_t`) | `semanage fcontext -a -t CORRECT_t "PATH(/.*)?"` and `restorecon -Rv PATH` |
| `name_connect` or `name_bind` on a **port** | Port label: `semanage port -a -t TYPE_port_t -p tcp PORT`, or a boolean (`httpd_can_network_connect`) |
| `audit2why` names a **boolean** | `setsebool -P BOOLEAN on` |
| Process is in the **wrong domain** (for example `unconfined_service_t`, `init_t`) | Label the executable (`bin_t` / its proper type), or write a policy |
| Genuinely missing rule for custom software | A reviewed custom module (see previous page) |

**5. Verify and clean up.**

```console
# ausearch -m AVC -ts recent                           # no new denials after the fix
# semanage permissive -d httpd_t                       # remove temporary permissive domains
# semanage fcontext -l -C; semanage boolean -l -C; semanage port -l -C      # record what you changed
```

### Common cases

**Web content outside `/var/www`:**

```console
# semanage fcontext -a -t httpd_sys_content_t "/srv/site(/.*)?"
# restorecon -Rv /srv/site
```

**Web application uploads:**

```console
# semanage fcontext -a -t httpd_sys_rw_content_t "/srv/site/uploads(/.*)?"
# restorecon -Rv /srv/site/uploads
```

**Web server connects to a backend or API:** `setsebool -P httpd_can_network_connect on`.

**Database datadir moved:**

```console
# semanage fcontext -a -e /var/lib/mysql /data/mysql
# restorecon -Rv /data/mysql
```

**SSH on a new port:** `semanage port -a -t ssh_port_t -p tcp 2222`.

**Files moved with `mv` have wrong labels:** `restorecon -Rv /destination`.

**Service in `/opt` is not confined or is denied:** label binaries (`bin_t`), configuration (`etc_t`), logs (`var_log_t`), and data (`var_lib_t`) correctly, or use `sepolicy generate` for a proper policy.

**NFS home directories:** `setsebool -P use_nfs_home_dirs on`.

**Container volume denied:** use `:z` or `:Z` on the volume, or label the directory `container_file_t`.

**Cron job or script denied:** see which domain `crond_t` runs it in, and the label of the script. Scripts in unusual places run as `unconfined_service_t` or are blocked.

### When there is no AVC but it still fails

- **`dontaudit` rules** hide some denials: `semodule -DB`, reproduce, review `ausearch`, then **`semodule -B`** to restore.
- **`auditd` stopped or the log full.** Check `systemctl status auditd` and the free space in `/var/log/audit`.
- **Not SELinux.** Check file permissions, `namei -l`, firewall, `sudo`, and the application's own logs.
- **Denied at the process level** (`class process`): `execmem`, `execstack`, `setcap`. These are booleans such as `deny_execmem` or `selinuxuser_execmod`.

### Relabeling the whole system

```console
# touch /.autorelabel && reboot            # full relabel at next boot (long on large disks)
# fixfiles -F onboot                       # same idea
# restorecon -Rv /var                      # targeted relabel
# fixfiles check /srv                      # report mislabeled files without changing
```

### Useful quick checks

```console
$ ls -Z FILE; ps -eZ | grep PROC          # source and target contexts
$ id -Z
$ getsebool -a | grep -i KEYWORD
$ sesearch -A -s httpd_t -t TARGET_t      # is there any rule?
$ sestatus -b | head -30                  # all booleans with values
```

## Common pitfalls

- **Disabling SELinux** instead of reading the denial. Spend five minutes on `audit2why`.
- **Staying in permissive mode** after troubleshooting. Set it back immediately.
- **Applying a generated module** without reading it.
- **Treating the symptom** (allowing `httpd_t` to read `user_home_t`) instead of the cause (content in the wrong place or label).
- **Assuming the first denial is the only one.** Fix one, test, and repeat. In permissive mode you see the whole set at once.
- **Ignoring the `permissive=1` flag** in denials. The system was not enforcing at that time.
- **Forgetting `-P`** on booleans and **forgetting `restorecon`** after `fcontext`.
- **Running `restorecon` on a data path while the service is writing** large amounts of data (I/O load). Do it in a maintenance window.
- **Not recording changes.** Keep the commands in configuration management so rebuilt servers get the same labels.

## Further reading

- [Red Hat: troubleshooting problems related to SELinux](https://docs.redhat.com)
- [`ausearch(8)`](https://man7.org/linux/man-pages/man8/ausearch.8.html)
- [`audit2why(8)`](https://man7.org/linux/man-pages/man8/audit2why.8.html)
- [`sealert`/setroubleshoot](https://github.com/fedora-selinux/setroubleshoot)
- [SELinux User's and Administrator's Guide](https://docs.redhat.com)
