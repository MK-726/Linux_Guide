# Security Logs and Event Analysis

## Overview

Attacks and mistakes leave traces. The challenge is knowing **which logs to read**, **which patterns indicate trouble**, and how to do it **at scale**. This page covers the security-relevant logs on Linux, typical indicators of compromise, and a triage approach. General log tools are in [Log Filtering and Analysis](../../part-2-system-administration/10-logging-time-and-scheduling/03-log-filtering-and-analysis.md).

## Key concepts

### Security-relevant log sources

| Source | Content | Location / command |
|--------|---------|--------------------|
| **Authentication log** | SSH, `sudo`, `su`, PAM, `login` | `/var/log/auth.log` (Debian-based), `/var/log/secure` (RHEL-based), `journalctl -t sshd -t sudo -t su` |
| **Login records** | Successful, failed, and last logins | `last`, `lastb`, `lastlog`, `/var/log/wtmp`, `btmp` |
| **auditd** | Syscalls, file watches, user commands, AVC denials | `/var/log/audit/audit.log`, `ausearch` |
| **systemd journal** | Services, kernel, boot | `journalctl` |
| **Kernel log** | Firewall drops (with log rules), module loads, AppArmor/SELinux denials, OOM | `journalctl -k`, `dmesg` |
| **Firewall logs** | Dropped and accepted connections | `nft`/`iptables` log rules, `firewalld` `--set-log-denied` |
| **Web and application logs** | Requests, errors, exploitation attempts | `/var/log/nginx/`, `/var/log/apache2/`, application logs |
| **Package logs** | Installs and removals | `/var/log/dpkg.log`, `/var/log/dnf.log`, `/var/log/apt/history.log` |
| **Cron and systemd timers** | Scheduled jobs | `journalctl -u cron`, `/var/log/cron` |
| **fail2ban** | Banned addresses | `/var/log/fail2ban.log`, `fail2ban-client status` |
| **Cloud and network** | VPC flow logs, cloud audit logs, DNS logs | Provider tools |

### What to look for: indicators of compromise

| Stage | Indicators |
|-------|------------|
| **Reconnaissance, brute force** | Many `Failed password` or `Invalid user` lines from one or many addresses, port scans in firewall logs, many 404/403 in web logs |
| **Initial access** | **Successful** login after many failures, login from an unusual country, time, or source, new SSH key use (fingerprint not seen before), web shells, exploit strings in web logs |
| **Execution** | Unexpected processes, shells spawned by web servers (`httpd` → `sh`), downloads with `curl`/`wget` to `/tmp`, base64-decoded commands |
| **Persistence** | New users or `authorized_keys` entries, new cron jobs or systemd units/timers, modified `~/.bashrc`/`/etc/profile.d`/`rc.local`, `/etc/ld.so.preload`, new SUID files, PAM changes |
| **Privilege escalation** | `sudo` use by unexpected users, new SUID/capability files, kernel exploit crashes, `su` to root |
| **Defense evasion** | Log files truncated or deleted, `history -c`, stopped `auditd`/`rsyslog`, modified time stamps, disabled SELinux/AppArmor, deleted binaries still running |
| **Lateral movement** | SSH from the server to other internal hosts, new internal connections, use of stored keys |
| **Collection and exfiltration** | Large outbound transfers, archives created in `/tmp`, connections to rare destinations or odd ports, DNS tunneling |
| **Impact** | Mass file modification or encryption, service stops, wiped disks |

(These map to the MITRE ATT&CK tactics.)

## Commands

### Authentication and logins

```console
# journalctl -u sshd --since "24 hours ago" | grep -E 'Failed password|Invalid user' | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
    842 203.0.113.9
    120 198.51.100.4
# grep 'Accepted' /var/log/auth.log | awk '{print $1,$2,$3,$9,$11}' | tail          # successful logins (Debian-based)
# journalctl -u sshd | grep 'Accepted publickey' | tail                              # with LogLevel VERBOSE: shows key fingerprints
# lastb -n 20 -i                       # failed logins with IPs (needs root)
# last -n 20 -i -F                     # successful logins with IPs and full times
# lastlog -t 30 | head                 # accounts that logged in during the last 30 days (-b 90 shows those NOT used for more than 90 days)
# who -a ; w                           # who is on right now
```

Find a **success after failures** from the same address (a sign of a guessed password):

```console
# for ip in $(grep 'Failed password' /var/log/auth.log | awk '{print $(NF-3)}' | sort -u); do grep -q "Accepted .* from $ip" /var/log/auth.log && echo "ALERT: success after failures from $ip"; done
```

### Privilege use

```console
# journalctl _COMM=sudo --since today | grep COMMAND
# grep -E 'sudo:.*(COMMAND|incorrect password|NOT in sudoers)' /var/log/auth.log | tail
# ausearch -m USER_CMD,USER_AUTH,USER_ACCT -ts today -i | tail
# ausearch -m ADD_USER,DEL_USER,ADD_GROUP,USER_MGMT -ts week -i          # account changes
# journalctl -t useradd -t usermod -t groupadd --since "7 days ago"
# getent group sudo wheel                                                # who has admin rights now
```

### Persistence checks

```console
# find /etc/cron* /var/spool/cron -type f -mtime -7 -ls 2>/dev/null           # recently changed cron files
# find /etc/systemd/system /usr/lib/systemd/system /root/.config/systemd -type f -mtime -7 -ls 2>/dev/null
# systemctl list-unit-files --state=enabled | head -50 ; systemctl list-timers --all
# find /home /root -name authorized_keys -mtime -30 -exec ls -l {} \;
# cat /etc/ld.so.preload 2>/dev/null                                           # should normally not exist
# find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -mtime -30 -ls 2>/dev/null     # new SUID/SGID files
# find /etc /usr/bin /usr/sbin /bin /sbin -type f -mtime -3 -ls 2>/dev/null     # system files changed recently
# grep -r -l -E 'curl|wget|nc |bash -i|/dev/tcp' /etc/cron* /etc/profile.d /home/*/.bashrc 2>/dev/null
# awk -F: '$3==0 {print $1}' /etc/passwd ; awk -F: '$2=="" {print $1}' /etc/shadow
```

### Processes and network

```console
# ps auxf --width=200 | less                                  # process tree: look for odd parents (web server → shell)
# ss -tnp state established | awk '{print $5, $6}' | sort | uniq -c | sort -rn | head      # outbound connections with processes
# ss -tulpn                                                    # new listeners
# ls -l /proc/*/exe 2>/dev/null | grep -E 'deleted|/tmp|/dev/shm'   # running from deleted or temp files
# lsof -nP -i | head -40
# lsmod | head -20 ; journalctl -k | grep -i -E 'module|taint'
# dmesg | grep -i -E 'segfault|oom|apparmor|denied|audit' | tail
```

### Web server logs (exploitation attempts)

```console
# awk '$9 ~ /^(4|5)/ {print $9, $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
# grep -i -E 'union.*select|\.\./|/etc/passwd|<script|%00|cmd=|wget%20|curl%20|jndi:' /var/log/nginx/access.log | head
# awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head           # top clients
```

### Evidence of log tampering

```console
# ls -l /var/log/ ; find /var/log -type f -size 0 -ls                # empty logs
# journalctl --verify
# ausearch -m CONFIG_CHANGE -i ; ausearch -m DAEMON_END,DAEMON_START -i
# last -x | grep -E 'reboot|shutdown' | head
# journalctl -u rsyslog -u auditd --since yesterday | grep -i -E 'stop|start|fail' | head
```

Gaps in time stamps, truncated files, and a stopped logging service are red flags. Keep **remote copies** of logs.

### fail2ban (automatic blocking)

```console
# apt install fail2ban             # or: dnf install fail2ban (EPEL)
# cat > /etc/fail2ban/jail.d/sshd.local <<'EOF'
[sshd]
enabled  = true
port     = ssh
maxretry = 5
findtime = 10m
bantime  = 1h
backend  = systemd
EOF
# systemctl enable --now fail2ban
# fail2ban-client status sshd
# fail2ban-client set sshd unbanip 203.0.113.9
```

fail2ban reduces noise and brute-force attempts. It is **not** a replacement for keys, MFA, and firewall limits. Be careful not to ban your own addresses (`ignoreip`).

### Initial incident triage (live system)

Collect **volatile** data first, then dig deeper. Work from a trusted shell, record every command and time, and avoid changing files.

```console
# date -u; hostname; uptime; who -a; last -n 30
# ps auxf > /safe/ps.txt; ss -tulpn > /safe/ss-listen.txt; ss -tnp > /safe/ss-conn.txt
# lsof -nP > /safe/lsof.txt; ip addr; ip route; ip neigh
# lsmod; mount; df -h; cat /proc/mounts
# crontab -l -u root; ls -la /etc/cron.*; systemctl list-units --type=service --state=running
# cp -a /var/log /safe/var-log-copy     # to external storage, not the compromised disk
```

Then decide: **contain** (isolate network, revoke credentials, snapshot the VM or disk for forensics), **eradicate** (rebuild from known-good sources, not "clean up"), **recover**, and **learn** (root cause, detection gap). Follow your organization's incident response plan and involve security/legal teams where required.

## Scaling up: central logging and detection

| Capability | Tools |
|------------|-------|
| **Collect and ship** | rsyslog, journald remote, Filebeat/Fluent Bit/Vector, Wazuh agent, auditd `audisp-remote` |
| **Store and search** | OpenSearch/Elasticsearch, Loki, Graylog, Splunk |
| **Detect (rules and alerts)** | Wazuh, Elastic Security, Sigma rules, Splunk ES, Falco (runtime behavior) |
| **Correlate with other data** | Cloud audit logs, network flow, identity provider logs, EDR |
| **Respond** | Ticketing/SOAR, automated blocks, on-call |

Start with a small set of **high-value alerts**: root/admin logins, new accounts or keys, `sudo` by non-admins, changes to critical files, service stops of logging/audit, and successful login after many failures. Add more as you learn what is normal.

## Common pitfalls

- **Logs only on the local host.** The attacker deletes them. Ship them off the system.
- **Unsynchronized clocks** make timelines impossible. Use NTP and UTC.
- **No baseline of normal.** You cannot spot the abnormal. Learn typical logins, jobs, and traffic first.
- **Alert fatigue.** Too many low-value alerts. Tune and prioritize.
- **Logging too little** (no `LogLevel VERBOSE` for SSH, no auditd) so investigations lack data.
- **Logging secrets** (passwords in URLs, tokens in command lines). Protect and sanitize.
- **Searching only for known strings.** Also look for **anomalies**: new, rare, or unusual events.
- **Modifying the evidence** (reboots, cleanup, `apt upgrade`) before capturing it, when forensics may be needed.
- **Assuming a clean `rpm -Va` means no compromise.** Attackers can alter the package database, kernel, or run in memory only.
- **Reacting without a plan.** Prepare an incident response procedure, contacts, and tools **before** you need them.
- **Retention too short.** Many intrusions are discovered months later. Keep security logs at least as long as your policy and regulations require (commonly 90 days to a year).

## Further reading

- [MITRE ATT&CK for Linux](https://attack.mitre.org/matrices/enterprise/linux/)
- [NIST SP 800-92: Guide to Computer Security Log Management](https://csrc.nist.gov/publications/detail/sp/800-92/final)
- [NIST SP 800-61: Incident Handling Guide](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final)
- [Wazuh documentation](https://documentation.wazuh.com)
- [Sigma rules](https://github.com/SigmaHQ/sigma)
- [fail2ban](https://github.com/fail2ban/fail2ban)
