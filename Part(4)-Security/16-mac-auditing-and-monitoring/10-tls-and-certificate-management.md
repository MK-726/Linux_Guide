# TLS and Certificate Management

## Overview

**TLS** (Transport Layer Security) encrypts and authenticates connections for HTTPS, mail, LDAP, databases, and many internal services. It relies on **X.509 certificates** that bind a **public key** to a **name**, signed by a **Certificate Authority (CA)**. Expired or misconfigured certificates cause outages and security failures, so certificate management is an operational discipline: create, deploy, renew, monitor, and replace. Cryptographic background is in [Cryptography Fundamentals](09-cryptography-fundamentals.md).

## Key concepts

### What a certificate contains

| Field | Meaning |
|-------|---------|
| **Subject** | Who the certificate is for (`CN=www.example.com`). The CN is legacy |
| **Subject Alternative Name (SAN)** | The **host names or IPs** the certificate is valid for. **Required** by modern clients |
| **Issuer** | The CA that signed it |
| **Validity** (`Not Before`, `Not After`) | Time range |
| **Public key** | The subject's key (RSA, ECDSA) |
| **Key usage / extended key usage** | Allowed purposes (`serverAuth`, `clientAuth`, `CA:TRUE`) |
| **Signature** | The CA's signature over all of this |
| **Serial number, extensions (CRL/OCSP URLs, SCTs)** | Management and revocation information |

### The chain of trust

```
Root CA (self-signed, in the system trust store)
   └── Intermediate CA (signed by the root)
          └── Leaf certificate  (www.example.com, signed by the intermediate)
```

- Clients trust the **root CAs** in their **trust store**.
- The server must send the **leaf plus intermediate(s)** (the "chain" or `fullchain`). The root is not sent.
- A client accepts a certificate if the chain leads to a trusted root, the name matches a SAN, the dates are valid, and the certificate is not revoked.

### Kinds of certificates and CAs

| Kind | Notes |
|------|-------|
| **Public CA** (Let's Encrypt, commercial CAs) | Trusted by browsers and systems. For internet-facing services |
| **Private / internal CA** | You run it and distribute its root to your clients. For internal services, mutual TLS, VPNs |
| **Self-signed** | Signs itself. Not trusted by default. Fine for local tests, not for production |
| **Wildcard** (`*.example.com`) | Covers one label level. A leaked key affects every subdomain |
| **Client certificate** | Authenticates a **user or machine** (mutual TLS) |

> [!NOTE]
> Public certificate lifetimes are getting **shorter**. The CA/Browser Forum has approved a schedule that reduces the maximum validity in steps through 2029, and Let's Encrypt already issues short-lived certificates. Plan for **automated renewal**. Manual renewals do not scale.

### Formats

| Format | Description | File names |
|--------|-------------|------------|
| **PEM** | Base64 text with `-----BEGIN ...-----` markers. Most common on Linux | `.pem`, `.crt`, `.cer`, `.key` |
| **DER** | Binary | `.der`, `.cer` |
| **PKCS#12 / PFX** | Binary bundle with key, certificate, and chain, password protected | `.p12`, `.pfx` |
| **JKS / PKCS#12** | Java keystores | `.jks` |
| **CSR** | Certificate Signing Request: your public key and name, sent to a CA | `.csr` |

### TLS versions and settings

| Version | Status |
|---------|--------|
| SSL 2/3, TLS 1.0, TLS 1.1 | **Disabled.** Insecure and deprecated |
| **TLS 1.2** | Acceptable with modern AEAD cipher suites |
| **TLS 1.3** | **Preferred.** Faster, simpler, forward secrecy always |

## Commands

### Create a key and a CSR

```console
$ openssl req -new -newkey rsa:3072 -nodes -keyout www.example.com.key -out www.example.com.csr \
      -subj "/C=US/O=Example Inc/CN=www.example.com" \
      -addext "subjectAltName=DNS:www.example.com,DNS:example.com"
$ openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -keyout ec.key -out ec.csr -subj "/CN=api.example.com" -addext "subjectAltName=DNS:api.example.com"
$ chmod 600 *.key
$ openssl req -in www.example.com.csr -noout -text -verify | head -20      # check the request
```

`-nodes` means the key is **not** encrypted with a passphrase (needed for unattended service restarts). Protect the file with permissions instead.

Send the CSR to the CA. Keep the **private key** on the server (never send it).

### Self-signed certificate (tests only)

```console
$ openssl req -x509 -newkey rsa:3072 -nodes -keyout test.key -out test.crt -days 365 \
      -subj "/CN=test.local" -addext "subjectAltName=DNS:test.local,IP:192.0.2.10"
```

### A small private CA (lab and internal use)

```console
# 1. Create the CA key and root certificate (guard ca.key carefully; offline if possible)
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-384 -out ca.key
$ chmod 600 ca.key
$ openssl req -x509 -new -key ca.key -sha384 -days 3650 -subj "/O=Example Inc/CN=Example Internal Root CA" \
      -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign" -out ca.crt

# 2. Sign a server request with the CA
$ cat > server.ext <<'EOF'
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:app.internal.example.com,IP:10.0.0.20
EOF
$ openssl x509 -req -in app.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out app.crt -days 365 -sha256 -extfile server.ext
```

For production private CAs use a tool with policy and automation: **step-ca (Smallstep)**, **HashiCorp Vault / OpenBao PKI**, **EJBCA**, **cfssl**, **FreeIPA's CA**, or a cloud private CA. Use an **offline root** with **intermediate** CAs issuing certificates, short lifetimes, and automated renewal.

### Inspect certificates and chains

```console
$ openssl x509 -in cert.pem -noout -subject -issuer -dates -serial -fingerprint -sha256
$ openssl x509 -in cert.pem -noout -ext subjectAltName
$ openssl x509 -in cert.pem -noout -text | less
$ openssl x509 -in cert.pem -noout -checkend $((30*86400)) && echo "valid for 30+ days" || echo "expires within 30 days!"

# check what a server presents
$ openssl s_client -connect www.example.com:443 -servername www.example.com -showcerts </dev/null 2>/dev/null
$ echo | openssl s_client -connect www.example.com:443 -servername www.example.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates
$ openssl s_client -connect www.example.com:443 -servername www.example.com -tls1_2 </dev/null     # test a version
$ openssl s_client -starttls smtp -connect mail.example.com:587 </dev/null                          # STARTTLS protocols

# verify a chain
$ openssl verify -CAfile ca.crt -untrusted intermediate.pem leaf.pem
leaf.pem: OK

# does the key match the certificate? (the two hashes must be identical)
$ openssl x509 -in cert.pem -noout -pubkey | openssl sha256
$ openssl pkey -in private.key -pubout | openssl sha256

# bundle for a service that wants a chain file
$ cat leaf.pem intermediate.pem > fullchain.pem
```

### Convert formats

```console
$ openssl x509 -in cert.pem -outform der -out cert.der
$ openssl x509 -inform der -in cert.der -out cert.pem
$ openssl pkcs12 -export -out bundle.p12 -inkey private.key -in leaf.pem -certfile intermediate.pem
$ openssl pkcs12 -in bundle.p12 -nodes -out all.pem          # extract everything (protect the output!)
$ keytool -importkeystore -srckeystore bundle.p12 -srcstoretype PKCS12 -destkeystore keystore.jks
```

### Let's Encrypt and the ACME protocol

**ACME** automates issuance and renewal. Let's Encrypt is free. Clients: **certbot**, **acme.sh**, **lego**, and built-in support in **Caddy** and **Traefik**.

```console
# apt install certbot python3-certbot-nginx           # or: dnf install certbot python3-certbot-nginx
# certbot --nginx -d example.com -d www.example.com                    # obtain and configure Nginx
# certbot certonly --webroot -w /var/www/html -d example.com           # obtain only (HTTP-01 challenge)
# certbot certonly --manual --preferred-challenges dns -d '*.example.com'   # wildcard (DNS-01; use a DNS plugin to automate)
# certbot certificates                                                 # list and show expiry
# certbot renew --dry-run                                              # test renewal
# systemctl list-timers | grep certbot                                 # automatic renewal timer (or cron)
# certbot renew --deploy-hook "systemctl reload nginx"                 # reload services after renewal
```

| Challenge | How it proves control | Notes |
|-----------|----------------------|-------|
| **HTTP-01** | A file under `/.well-known/acme-challenge/` on port 80 | Simplest. Port 80 must be reachable. No wildcards |
| **DNS-01** | A TXT record `_acme-challenge.example.com` | Works for **wildcards** and for servers without public HTTP. Automate with DNS provider API tokens (protect the token) |
| **TLS-ALPN-01** | A special TLS handshake on port 443 | Used by some tools |

Files are in `/etc/letsencrypt/live/example.com/`: `fullchain.pem` (certificate plus chain, use this in servers), `privkey.pem` (private key, root-only).

### Install a certificate in a service (Nginx example)

```
server {
    listen 443 ssl;
    http2 on;
    server_name www.example.com;

    ssl_certificate     /etc/letsencrypt/live/www.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000" always;
}
```

Use the **Mozilla SSL Configuration Generator** for current cipher suites for Nginx, Apache, HAProxy, and others. Validate and reload: `nginx -t && systemctl reload nginx`.

### Trust stores (making a private CA trusted)

| Family | Add the CA certificate | Update command |
|--------|------------------------|----------------|
| Debian-based | `/usr/local/share/ca-certificates/corp-ca.crt` (PEM, `.crt` extension) | `update-ca-certificates` |
| RHEL-based | `/etc/pki/ca-trust/source/anchors/corp-ca.pem` | `update-ca-trust` |
| SUSE-based | `/etc/pki/trust/anchors/corp-ca.pem` | `update-ca-certificates` |

The result is the system bundle: `/etc/ssl/certs/ca-certificates.crt` (Debian-based) or `/etc/pki/tls/certs/ca-bundle.crt` (RHEL-based).

Applications with their own stores need separate steps:

| Application | How |
|-------------|-----|
| Java | `keytool -importcert -cacerts -alias corp-ca -file corp-ca.pem` |
| Python `requests` | `REQUESTS_CA_BUNDLE=/path/ca.pem` (or `SSL_CERT_FILE`) |
| Node.js | `NODE_EXTRA_CA_CERTS=/path/ca.pem` |
| `curl` | `--cacert file` or the system bundle |
| Firefox, containers, language runtimes | Their own certificate settings |

### Mutual TLS (client certificates)

Both sides authenticate with certificates. Useful for service-to-service, APIs, and admin interfaces.

```
# Nginx: require client certificates signed by your CA
ssl_client_certificate /etc/ssl/corp-ca.crt;
ssl_verify_client on;
```

```console
$ curl --cert client.crt --key client.key --cacert ca.crt https://api.internal.example.com/
```

### Monitor expiry (very important)

```console
$ for h in www.example.com api.example.com; do
    echo -n "$h: "; echo | openssl s_client -connect $h:443 -servername $h 2>/dev/null | openssl x509 -noout -enddate; done
$ certbot certificates
```

Use monitoring (Prometheus `blackbox_exporter` or `x509-certificate-exporter`, Nagios `check_http -C 30`, `ssl-cert-check`, cloud monitors) to **alert at 30, 14, and 7 days** before expiry, and on **renewal failures**. Also monitor Certificate Transparency logs (crt.sh) for unexpected certificates issued for your domains.

### Test the TLS configuration

```console
$ testssl.sh https://www.example.com                 # comprehensive local scanner
$ nmap --script ssl-enum-ciphers -p 443 www.example.com
$ sslscan www.example.com
$ curl -vI --tlsv1.3 https://www.example.com
```

Public services can also use SSL Labs (Qualys) tests.

### Common errors

| Error | Cause and fix |
|-------|---------------|
| `certificate has expired` | Renew. Check the **system clock** on client and server |
| `unable to get local issuer certificate` / `unable to verify the first certificate` | The server does not send the **intermediate**. Use `fullchain.pem` |
| `self-signed certificate` / `self-signed certificate in certificate chain` | The CA is not trusted on the client. Install the CA in the trust store |
| `hostname mismatch` / `certificate is not valid for ...` | The name is not in the **SAN**. Reissue with the right names (or use the correct host name) |
| `key values mismatch` | Private key does not match the certificate. Compare public key hashes |
| `bad certificate` / handshake failure on mTLS | Client certificate missing, expired, wrong CA, or wrong `extendedKeyUsage` |
| `no shared cipher` / `protocol version` | Client and server have no common TLS version or cipher. Check `ssl_protocols` and client age |
| Certificate OK in browser, fails in `curl`/Java/Python | Different trust store or missing intermediate |
| Works after reload only | The service has not reloaded the renewed certificate. Add a deploy hook |

## Common pitfalls

- **Forgetting renewal** (the classic outage). Automate it and monitor expiry.
- **Renewing but not reloading** the service, which keeps serving the old certificate.
- **Missing intermediate certificates** (`cert.pem` instead of `fullchain.pem`).
- **Certificates without SAN.** Modern clients reject them.
- **Private keys world-readable**, committed to Git, copied between servers, or reused across many hosts. Restrict (`600`, owner root or the service group) and keep one key per service where possible.
- **Wildcard certificates** spread across many systems. One compromise exposes all.
- **Disabling verification** (`curl -k`, `verify=False`, `NODE_TLS_REJECT_UNAUTHORIZED=0`) permanently.
- **Weak protocol settings** (TLS 1.0/1.1, old ciphers) left from old configurations.
- **Poorly protected private CA keys.** Anyone with the CA key can impersonate any internal service. Use offline roots, HSMs, and short-lived certificates.
- **Not planning revocation and rotation.** Know how to replace a key quickly after a compromise.
- **Unsynchronized clocks.** Certificates appear "not yet valid" or "expired".
- **Port 80 blocked** when using HTTP-01, so renewals fail silently.
- **DNS API tokens** for DNS-01 stored with excessive permissions. Scope them to the challenge zone.

## Further reading

- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org)
- [`openssl-req(1)` and `openssl-x509(1)`](https://docs.openssl.org/master/man1/openssl-req/)
- [Let's Encrypt documentation](https://letsencrypt.org/docs/)
- [Certbot](https://certbot.eff.org)
- [Smallstep: certificates and PKI](https://smallstep.com/docs/)
- [RFC 8446: TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 8555: ACME](https://datatracker.ietf.org/doc/html/rfc8555)
