# 15:30 — nginx2root (Xint) applicability check

## Input
User asked: can we use "nginx2root xint" vuln (disclosed 2026-11-18, 5 days ago)?
5 nginx CVEs + kernel SCTP LPE chain, by Xint. Fixed nginx 1.30.7/1.31.9.
Affected-version range: 1.9.5–1.31.8 (distro backports aside).

## Box facts (via fresh www-data shell, hktr5 account)
```
nginx version: nginx/1.18.0 (Ubuntu)
dpkg: 1.18.0-6ubuntu14.3        # 2022-era pkg, NO Oct-2026 USN backports → in vuln range
/etc/nginx/nginx.conf + sites-enabled/default (complete config):
  listen 80 (default_server + 2million.htb vhost)
  server_tokens off
  php via unix socket /run/php/php-fpm.sock
  NO: ssl, http2, http3/quic, resolver, proxy_pass, mirror, mail{} block
External checks: no h2 prior-knowledge, no h2c upgrade, 443/tcp closed, 443/udp (QUIC) no reply
```

## CVE-by-CVE verdict
| CVE | Requirement | Box | Usable |
|---|---|---|---|
| 2026-123007 | HTTP/2 listener | none | no |
| 2026-123633 | HTTP/2 listener | none | no |
| 2026-121810 | HTTP/3 QUIC listener | none | no |
| 2026-123634 | resolver + request-derived proxy_pass + mirror | none | no |
| 2026-123425 | mail proxy + auth_http | none | no |

Kernel SCTP LPE (6th piece): would chain AFTER an nginx RCE; moot. Our root path
stays CVE-2023-0386 (OverlayFS), independent.

## Conclusion
Binary is vulnerable-version (no backport) but configuration exposes NONE of the five
code paths. Plain HTTP/1.1 + fastcgi-only vhost = no attack surface for nginx2root.
Not usable on this box.
