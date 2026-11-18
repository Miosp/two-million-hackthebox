Input:
```sh
IP=10.129.229.66
curl -sI -H "Host: 2million.htb" http://$IP/
curl -s -D - -o /dev/null -H "Host: 2million.htb" http://$IP/doesnotexist12345
curl -s -D - -o /dev/null -X BOGUS -H "Host: 2million.htb" http://$IP/
printf 'GARBAGE\r\n\r\n' | timeout 5 nc $IP 80
curl -s -D - -o /dev/null -H "Host: 2million.htb" http://$IP/api/v1
curl -s -D - -o /dev/null -H "Host: 2million.htb" http://$IP/login
curl -s -D - -o /dev/null -X OPTIONS -H "Host: 2million.htb" http://$IP/
```

Output:
```
---HEAD /---  (also GET / does the same)
HTTP/1.1 405 Method Not Allowed
Server: nginx
Content-Type: text/html; charset=UTF-8
Set-Cookie: PHPSESSID=s1ro4dpfrcaeg7u6ba5sc48r0o; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache

---404 / nonexistent path---
HTTP/1.1 301 Moved Permanently
Server: nginx
Content-Length: 162
Location: http://2million.htb/404
(all nonexistent paths 301-redirect to /404 — catch-all)

---bogus method BOGUS /---
HTTP/1.1 405 Not Allowed
Server: nginx
Content-Type: text/html; charset=utf-8
Content-Length: 150

---raw garbage request (nc)---
HTTP/1.1 400 Bad Request
Server: nginx
Content-Type: text/html; charset=utf-8
Content-Length: 150
Connection: close
<html>... <hr><center>nginx</center> ...</html>

---/api/v1---
HTTP/1.1 401 Unauthorized
Server: nginx
Set-Cookie: PHPSESSID=78o8vjmfec3v3v9st8smfu6erd; path=/
Transfer-Encoding: chunked

---/login---
HTTP/1.1 200 OK
Server: nginx
Set-Cookie: PHPSESSID=nc2bo5765eebahs07tk3pe8ch6; path=/

---OPTIONS /---
HTTP/1.1 405 Not Allowed (no Allow header)
```

Notes:
- `Server: nginx` everywhere — version stripped even on nginx-generated 400 page => `server_tokens off`.
- PHPSESSID => PHP backend. No X-Powered-By => `expose_php off` or stripped by nginx.
- OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 => Ubuntu 22.04 jammy. Stock nginx there = 1.18.0.
