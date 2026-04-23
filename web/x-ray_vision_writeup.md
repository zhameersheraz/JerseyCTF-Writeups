# X-Ray Vision — JerseyCTF Writeup

**Challenge:** X-Ray Vision  
**Category:** web  
**Difficulty:** Easy  
**Points:** 442  
**Solves:** 55  
**Challenge Author:** Rayyan Khan  
**Writeup by:** zham  
**Flag:** `jctf{r0t_y0ur_w4y_t0_4cc3ss}`

---

## Description

> X-Ray Vision's internal employee portal was accidentally pushed to staging with debug artifacts left behind. The developer API is locked down, but someone forgot to clean up before deploying. The credential is in there somewhere — but it won't be handed to you in plaintext.

**URL:** `http://x-ray-vision.aws.jerseyctf.com`

---

## Background Knowledge (Read This First!)

### What are HTML data attributes?

HTML elements can carry arbitrary extra data using `data-*` attributes. They are invisible on the rendered page but fully readable in the page source or DevTools. Developers sometimes use them during debugging to embed credentials, tokens, or notes — and forget to remove them before going to production.

For example:
```html
<div data-token="abc123" style="display:none"></div>
```

Anyone who views the source can read `abc123`.

### What is ROT-N (Caesar Cipher)?

**ROT-N** shifts every letter in a string by N positions forward in the alphabet. **ROT13** (shift by 13) is the most common variant because applying it twice returns the original string (13 + 13 = 26 = full alphabet).

For example:
```
Input:  q3i3y0c3e_g00y5
ROT13:  d3v3l0p3r_t00l5
```

Non-letter characters (numbers, underscores) are left unchanged.

### What is a custom HTTP request header?

HTTP requests can include extra headers to pass metadata to the server — things like authentication tokens, API keys, or content types. A custom header like `x-secret-token` is non-standard and used by the application itself to gate access to restricted endpoints.

You can send custom headers from the browser Console using the `fetch` API:

```javascript
fetch('/api/status', {
  headers: { 'x-secret-token': 'myvalue' }
}).then(r => r.text()).then(console.log)
```

---

## Solution — Step by Step

### Step 1 — View the Page Source

Open the site and view the page source (Ctrl+U) or open DevTools → Sources → `(index)`.

Near the bottom of the HTML, hidden with `display:none`, is a div left over from debugging:

```html
<div id="sys-cache"
     style="display:none"
     data-stage-note="remove before prod"
     data-h="x-secret-token"
     data-t="q3i3y0c3e_g00y5">
</div>
```

Two important values are sitting here:

- `data-h` — the name of the secret HTTP header: `x-secret-token`
- `data-t` — the token value: `q3i3y0c3e_g00y5`

The challenge description warns the credential is *"not handed to you in plaintext"* — so `q3i3y0c3e_g00y5` is encoded.

### Step 2 — Decode the Token (ROT13)

The token `q3i3y0c3e_g00y5` looks like ROT-encoded text. Try all 25 ROT shifts:

```javascript
const s = 'q3i3y0c3e_g00y5';
for (let n = 1; n <= 25; n++) {
  const r = s.replace(/[a-z]/gi, c => {
    const base = c >= 'a' ? 97 : 65;
    return String.fromCharCode((c.charCodeAt(0) - base + n) % 26 + base);
  });
  console.log(n, r);
}
```

ROT13 (shift 13) gives:

```
d3v3l0p3r_t00l5
```

That reads as `developer_tools` — the decoded secret token is `d3v3l0p3r_t00l5`.

### Step 3 — Call the API with the Token

Open DevTools (F12) → **Console** tab on the site. Paste and run:

```javascript
fetch('/api/status', {
  headers: { 'x-secret-token': 'd3v3l0p3r_t00l5' }
}).then(r => r.text()).then(console.log)
```

The server responds with:

```json
{"status": "success", "flag": "jctf{r0t_y0ur_w4y_t0_4cc3ss}"}
```

---

## Flag

```
jctf{r0t_y0ur_w4y_t0_4cc3ss}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Browser DevTools — Sources / Page Source | Find the hidden `data-*` attributes in the HTML | ⭐ Easy |
| ROT13 / brute-force ROT decoder | Decode `q3i3y0c3e_g00y5` → `d3v3l0p3r_t00l5` | ⭐ Easy |
| Browser DevTools — Console tab | Send the `fetch()` request with the custom header | ⭐ Easy |

---

## Key Takeaways

- **"display:none" does not hide data.** Anything in the HTML source is readable, no matter how it is styled. Always strip debug artifacts before deploying.
- **`data-*` attributes are public.** They are designed for JavaScript to read — which means anyone can read them too.
- **ROT encoding is not encryption.** It is a trivially reversible substitution and provides zero real security.
- **Custom headers are still just strings.** If you can find the expected header name and value in the source, you can replay them with `fetch()` in the browser Console directly — no special tools needed.
