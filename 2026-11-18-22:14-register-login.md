Input:
```sh
IP=10.129.229.66
# register (fresh single-use invite code each time)
curl -X POST -H "Host: 2million.htb" -d "code=<FRESH>" -d "username=hktr428" -d "email=hktr428@2million.htb" \
  -d "password=Pw2million!HTB" -d "password_confirmation=Pw2million!HTB" http://$IP/api/v1/user/register
curl -X POST -H "Host: 2million.htb" -c tmp/cookies.txt -d "email=hktr428@2million.htb" -d "password=Pw2million!HTB" http://$IP/api/v1/user/login
```

Output:
```
register (1st, with pre-verified/burned code): 302 -> /register?error=Code+is+invalid!
register (2nd, fresh code):                    302 -> /register?error=Username+or+email+already+exists
   (=> 1st register DID succeed; error only on duplicate)
login username=hktr428&password=...:           302 -> /login?error=Please+enter+an+email
login email=hktr428@2million.htb&password=...: 302 -> /home  (auth OK, PHPSESSID set)
GET /home with session:                        200 18329B

# hktr429 (password=QNKCD7, magic-hash md5 0e830400971418735364996746983040):
login email=hktr429@2million.htb password=QNKCD7 (correct):   302 -> /home
login same email password=240610708 (other 0e magic hash):    302 -> /login?error=User+not+found
login same email password=WRONGWRONG (control):               302 -> /login?error=User+not+found
```

Notes:
- Login field is EMAIL, not username.
- Wrong password => "User not found" (not "wrong password") — suggests combined
  SELECT ... WHERE email=? AND password=md5(?) or generic error. Magic-hash (0e) loose
  comparison DOES NOT bypass => hashes compared strictly or SQL-side string compare.
