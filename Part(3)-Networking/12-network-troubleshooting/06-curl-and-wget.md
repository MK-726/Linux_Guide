# curl and wget

## Overview

`curl` and `wget` talk to web servers (HTTP, HTTPS, FTP). They are daily tools for **testing services**, **checking certificates and headers**, **measuring timing**, and **downloading files**. `curl` is the more powerful tool for testing. `wget` is convenient for downloads and mirroring.

## Commands

### curl: the essentials

```console
$ curl https://www.example.com                       # print the response body
$ curl -I https://www.example.com                    # headers only (HEAD request)
$ curl -s -o /dev/null -w '%{http_code}\n' https://www.example.com    # only the status code
$ curl -sS https://www.example.com/api/health        # silent, but show errors
$ curl -L http://example.com                         # follow redirects
$ curl -o page.html https://www.example.com          # save to a named file
$ curl -O https://example.com/file.tar.gz            # save with the remote name
$ curl -fsSL https://example.com/file -o file        # fail on HTTP errors, silent, follow redirects
```

| Option | Meaning |
|--------|---------|
| `-v` | **Verbose**: connection, TLS handshake, request and response headers |
| `-I` | Only headers (HEAD) |
| `-i` | Include response headers in the output |
| `-s` / `-S` | Silent / show errors even when silent |
| `-L` | Follow redirects |
| `-o FILE` / `-O` | Write to a file / use the remote file name |
| `-f` | Fail (non-zero exit) on HTTP errors (400+) |
| `-X METHOD` | HTTP method (`POST`, `PUT`, `DELETE`) |
| `-H "Name: value"` | Add a header |
| `-d 'data'` | Send POST data |
| `--json '{"a":1}'` | Send JSON with the right headers (curl 7.82+) |
| `-u user:pass` | Basic authentication |
| `-k` | **Skip certificate verification** (insecure) |
| `--cacert FILE` | Trust a specific CA |
| `--connect-timeout N` | Seconds to wait for the connection |
| `-m N` / `--max-time N` | Total time limit |
| `--retry N` | Retry on transient errors |
| `-4` / `-6` | Force IPv4 / IPv6 |
| `--resolve HOST:PORT:IP` | Use this IP for the name (test a server before DNS changes) |
| `--interface IFACE` | Use a specific interface or source address |
| `-x URL` | Use a proxy |
| `--http1.1`, `--http2`, `--http3` | Force a protocol version |
| `-w FORMAT` | Print timing and other information after the request |

### Verbose output: what to look for

```console
$ curl -v https://www.example.com -o /dev/null
*   Trying 203.0.113.80:443...
* Connected to www.example.com (203.0.113.80) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: CN=www.example.com
*  start date: Jan 10 00:00:00 2025 GMT
*  expire date: Apr 10 23:59:59 2025 GMT
*  issuer: C=US; O=Example CA; CN=Example CA 1
* SSL certificate verify ok.
> GET / HTTP/2
> Host: www.example.com
> User-Agent: curl/8.5.0
< HTTP/2 200
< content-type: text/html
```

Lines starting with `*` are connection details, `>` is what you send, and `<` is what you receive. You can see: which IP was used, whether TLS works, certificate dates and issuer, and the response status.

### Timing breakdown (find where time is spent)

```console
$ curl -s -o /dev/null -w 'dns: %{time_namelookup}s\nconnect: %{time_connect}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\ntotal: %{time_total}s\nstatus: %{http_code}\nip: %{remote_ip}\n' https://www.example.com
dns: 0.024s
connect: 0.037s
tls: 0.071s
ttfb: 0.182s
total: 0.190s
status: 200
ip: 203.0.113.80
```

| Value | Meaning | If slow, look at |
|-------|---------|------------------|
| `time_namelookup` | DNS lookup | Resolver, search domains, IPv6 |
| `time_connect` | TCP connect (cumulative) | Network latency, firewall, SYN loss |
| `time_appconnect` | TLS handshake complete (cumulative) | TLS, certificate checks, OCSP |
| `time_starttransfer` | First byte received (**TTFB**) | **Server processing time** (application, database) |
| `time_total` | Everything | Download size, bandwidth |

The **difference between values** (for example, `ttfb - tls`) is the time of each phase.

### Test a server directly, bypassing DNS

```console
$ curl --resolve www.example.com:443:198.51.100.20 https://www.example.com/ -I
$ curl -H 'Host: www.example.com' http://198.51.100.20/ -I          # virtual host on plain HTTP
```

Used to test a new server before changing DNS, or to test one backend behind a load balancer. Because of TLS (SNI), `--resolve` is better than `-H Host:` for HTTPS.

### API requests

```console
$ curl -X POST https://api.example.com/items \
       -H 'Content-Type: application/json' \
       -H "Authorization: Bearer $TOKEN" \
       -d '{"name":"test"}'
$ curl --json '{"name":"test"}' https://api.example.com/items
$ curl -sS https://api.example.com/items | jq .
```

Avoid putting secrets directly on the command line: they appear in shell history and in `ps`. Use an environment variable, `--netrc`, or a config file (`-K file`) with restricted permissions.

### Check a certificate

```console
$ curl -vI https://www.example.com 2>&1 | grep -E 'expire date|issuer|subject|verify'
$ echo | openssl s_client -connect www.example.com:443 -servername www.example.com 2>/dev/null | openssl x509 -noout -dates -subject -issuer
notBefore=Jan 10 00:00:00 2025 GMT
notAfter=Apr 10 23:59:59 2025 GMT
```

### wget

```console
$ wget https://example.com/file.tar.gz                # download
$ wget -O out.html https://example.com                # choose the output name
$ wget -c https://example.com/big.iso                 # resume an interrupted download
$ wget -q --spider https://example.com && echo up     # check the URL without downloading
$ wget -r -np -l 2 https://example.com/docs/          # recursive download, no parent, depth 2
$ wget --mirror --convert-links --page-requisites https://example.com
$ wget -t 3 -T 10 URL                                 # 3 tries, 10 s timeout
$ wget --header="Authorization: Bearer $TOKEN" URL
```

| Option | Meaning |
|--------|---------|
| `-O FILE` | Output file |
| `-c` | Continue a partial download |
| `-q` | Quiet |
| `--spider` | Check only |
| `-r`, `-np`, `-l N` | Recursive, no parent directories, depth |
| `--limit-rate=1m` | Limit bandwidth |
| `--no-check-certificate` | Skip certificate verification (**insecure**) |
| `-b` | Background |

### Verify downloads

```console
$ curl -fsSLO https://example.com/tool.tar.gz
$ curl -fsSLO https://example.com/tool.tar.gz.sha256
$ sha256sum -c tool.tar.gz.sha256
```

## Common pitfalls

- **Using `-k` or `--no-check-certificate` "to make it work".** It disables the protection TLS gives. Find the real cause (wrong clock, missing CA, wrong host name, expired certificate) and fix it.
- **`curl` returns exit code 0 on HTTP 500.** Use `-f` to treat HTTP errors as failures in scripts.
- **Forgetting `-L`.** You get a 301/302 page instead of the target.
- **Proxy environment variables** (`http_proxy`, `https_proxy`, `no_proxy`) change where requests go. Check `env | grep -i proxy`. Use `--noproxy '*'` to bypass.
- **Host header vs. SNI.** For HTTPS test with `--resolve`.
- **Quoting problems** with `&` in URLs. Quote the URL: `curl 'https://example.com/?a=1&b=2'`.
- **Secrets on the command line** (history, process list). Use files or variables.
- **`curl | sh` installers** run unreviewed code. Download, read, then run.
- **Measuring with the first request only.** Run several times. Caches and connection reuse change results.
- **Redirect loops or HSTS surprises.** Use `-v -L --max-redirs 5` to follow.

## Further reading

- [curl documentation](https://curl.se/docs/)
- [Everything curl (book)](https://everything.curl.dev)
- [`curl(1)`](https://curl.se/docs/manpage.html)
- [GNU Wget manual](https://www.gnu.org/software/wget/manual/wget.html)
