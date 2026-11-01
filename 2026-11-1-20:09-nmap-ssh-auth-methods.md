Input:
```sh
nmap -p 22 --script ssh-auth-methods 10.129.133.185
```

Output:
```
Starting Nmap 7.993 ( https://nmap.org ) at 2026-11-01 20:09 +0100
Nmap scan report for 2million.htb (10.129.133.185)
Host is up (0.030s latency).

PORT   STATE SERVICE
22/tcp open  ssh
| ssh-auth-methods:
|   Supported authentication methods:
|     publickey
|_    password

Nmap done: 1 IP address (1 host up) scanned in 0.30 seconds
```
