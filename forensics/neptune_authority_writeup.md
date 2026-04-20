# Neptune Authority — JerseyCTF Writeup

**Challenge:** Neptune Authority  
**Category:** forensics  
**Difficulty:** Medium  
**Points:** 734  
**Solves:** 32  
**Author:** Trent  
**Flag:** `jctf{48173926}`

---

## Description

> A network capture from Neptune's orbital defense perimeter shows the system entering escalation mode after the relay network reawakened. A shutdown authorization was transmitted over an encrypted channel before the perimeter locked down. Recover the materials needed to decrypt the exchange and stop the quarantine from closing around you.

**Attachment:** `neptune-defense.pcap`

---

## Background Knowledge (Read This First!)

### What is a PCAP file?

A **PCAP** (Packet Capture) file records all network traffic flowing through an interface at a specific point in time. It is the standard file format used by tools like Wireshark and tshark. Each "packet" in the file represents one unit of network data, and together they tell the full story of a network conversation.

### What is TLS?

**TLS** (Transport Layer Security) is the encryption protocol used by HTTPS. When a client connects to a server over TLS, they perform a **handshake** that establishes an encrypted session. After the handshake, all data is encrypted — including HTTP requests and responses — so you cannot read it directly from a packet capture.

However, if you have the **server's RSA private key**, you can decrypt TLS sessions that used RSA key exchange (non-ECDHE). tshark supports this natively.

### What is OpenSSL enc?

`openssl enc` is a command-line tool for symmetric encryption and decryption. When a file is encrypted with `openssl enc -aes-256-cbc`, it begins with the magic bytes `Salted__` followed by an 8-byte salt. The password and salt are combined to derive the actual encryption key.

You can decrypt with:
```bash
openssl enc -d -aes-256-cbc -in file.enc -out file.dec -pass pass:PASSWORD
```

### What is tshark?

**tshark** is the command-line version of Wireshark. It can parse pcap files, filter traffic by protocol, extract HTTP objects, follow TCP/TLS streams, and even decrypt TLS sessions when given a private key.

---

## Solution — Step by Step

### Step 1 — Survey the capture

Start by understanding what protocols and hosts are in the pcap:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r neptune-defense.pcap -q -z io,phs 2>/dev/null

Protocol Hierarchy Statistics
eth                frames:1733 bytes:236258
  ip               frames:1687 bytes:232796
    tcp            frames:1249 bytes:189290
      http         frames:220 bytes:76076
      tls          frames:63  bytes:23066
    icmp           frames:432 bytes:42336
```

Three key protocols: **HTTP** (unencrypted), **TLS** (encrypted), and **ICMP** (ping). The HTTP and TLS traffic are the most interesting.

Identify the hosts:

| IP | Role |
|----|------|
| 10.20.0.10 | Client |
| 10.20.0.50 | TLS server (port 8443) |
| 10.20.0.99 | HTTP server (port 8080) |

### Step 2 — Find the HTTP downloads

List all unique HTTP endpoints accessed:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r neptune-defense.pcap -Y "http" \
  -T fields -e http.request.uri -e http.response.code 2>/dev/null | sort -u

/ods.crt.enc    (GET)
/ods.key.enc    (GET)
/status         (GET → 404)
```

The client repeatedly polls `/status` (getting 404 each time), and eventually successfully downloads `/ods.crt.enc` and `/ods.key.enc`. These sound like an encrypted TLS certificate and private key.

Export them from the pcap:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r neptune-defense.pcap --export-objects http,http_objects 2>/dev/null

┌──(zham㉿kali)-[~]
└─$ file http_objects/ods.crt.enc http_objects/ods.key.enc
http_objects/ods.crt.enc: openssl enc'd data with salted password
http_objects/ods.key.enc: openssl enc'd data with salted password
```

Both files are symmetric-encrypted with OpenSSL. We need the password.

### Step 3 — Find the decryption password in HTTP headers

Examine the HTTP response headers for the `.enc` file downloads carefully:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r neptune-defense.pcap -Y "frame.number>=1258 and frame.number<=1265" -V 2>/dev/null \
  | grep -E "HTTP|X-|Content|Server"

GET /ods.crt.enc HTTP/1.1
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.25
Content-type: application/octet-stream
Content-Length: 1264
X-Orbit-Note: oldorbit
```

The custom header `X-Orbit-Note: oldorbit` is hidden in the response. This is the password.

### Step 4 — Decrypt the certificate and private key

```
┌──(zham㉿kali)-[~]
└─$ openssl enc -d -aes-256-cbc \
  -in http_objects/ods.crt.enc \
  -out ods.crt \
  -pass pass:oldorbit

*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.

┌──(zham㉿kali)-[~]
└─$ openssl enc -d -aes-256-cbc \
  -in http_objects/ods.key.enc \
  -out ods.key \
  -pass pass:oldorbit

┌──(zham㉿kali)-[~]
└─$ file ods.crt ods.key
ods.crt: PEM certificate
ods.key: OpenSSH private key (no password)
```

The certificate is issued to `CN=ods-nep-perimeter, O=OrbitalDefense`. The private key matches the TLS server at `10.20.0.50:8443`.

### Step 5 — Identify the TLS stream to decrypt

Check which TLS TCP streams have actual application data (not just handshakes):

```
┌──(zham㉿kali)-[~]
└─$ tshark -r neptune-defense.pcap \
  -Y "tls and ip.addr==10.20.0.50 and tls.record.content_type==23" \
  -T fields -e tcp.stream -e ip.src -e frame.len 2>/dev/null

71   10.20.0.50   235
71   10.20.0.10   203
```

TCP stream **71** is the only one with encrypted application data exchanged in both directions — this is the target.

### Step 6 — Decrypt the TLS stream with the private key

tshark can decrypt TLS sessions using the RSA private key via the `tls.keys_list` option:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r neptune-defense.pcap \
  -o "tls.keys_list:10.20.0.50,8443,http,ods.key" \
  -z "follow,tls,ascii,71" 2>/dev/null | grep -A 30 "Node 0:"

Node 0: 10.20.0.10:59960
Node 1: 10.20.0.50:8443

    113
HTTP/1.1 200 OK
Content-Type: text/plain

STATUS: ESCALATION_ACTIVE
COUNTDOWN: ACTIVE
SHUTDOWN_CODE: 48173926

    92
GET / HTTP/1.1
Host: ods-nep-perimeter
User-Agent: station-core/1.0
Connection: close
```

The decrypted HTTP response contains the shutdown authorization code.

🏁 **Flag: `jctf{48173926}`**

---

## One-liner Solve Script

Open a new file with nano:

```
┌──(zham㉿kali)-[~]
└─$ nano solve.sh
```

Paste the following script inside nano:

```bash
#!/bin/bash
# Step 1: Export HTTP objects from pcap
tshark -r neptune-defense.pcap --export-objects http,http_objects 2>/dev/null

# Step 2: Decrypt the key files (password found in X-Orbit-Note header)
openssl enc -d -aes-256-cbc -in http_objects/ods.crt.enc -out ods.crt -pass pass:oldorbit 2>/dev/null
openssl enc -d -aes-256-cbc -in http_objects/ods.key.enc -out ods.key -pass pass:oldorbit 2>/dev/null

# Step 3: Decrypt TLS stream and print flag
tshark -r neptune-defense.pcap \
  -o "tls.keys_list:10.20.0.50,8443,http,ods.key" \
  -z "follow,tls,ascii,71" 2>/dev/null | grep "SHUTDOWN_CODE"
```

Save and exit nano:

```
Ctrl + O      ← write/save the file
Enter         ← confirm the filename (solve.sh)
Ctrl + X      ← exit nano
```

Make it executable and run it:

```
┌──(zham㉿kali)-[~]
└─$ chmod +x solve.sh
┌──(zham㉿kali)-[~]
└─$ ./solve.sh
SHUTDOWN_CODE: 48173926
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `tshark` | Inspect pcap, filter protocols, export HTTP objects, decrypt TLS | ⭐⭐ Medium |
| `openssl enc -d` | Decrypt AES-256-CBC encrypted files with recovered password | ⭐ Easy |
| `grep` | Search HTTP response headers for the hidden password | ⭐ Easy |

---

## Key Takeaways

- **HTTP headers are often overlooked** — custom headers like `X-Orbit-Note` can contain credentials hidden in plain sight within a packet capture.
- **TLS is not unbreakable in forensics** — if the server's RSA private key is recoverable (e.g., downloaded from another channel in the same capture), all past TLS sessions using RSA key exchange can be decrypted.
- **Follow the chain** — the HTTP server gave you the files needed to decrypt the TLS server. Challenges often have a deliberate dependency chain between pieces.
- **`--export-objects` is your best friend** — tshark's object export saves every file transferred over HTTP in one command, far faster than reassembling TCP streams manually.
