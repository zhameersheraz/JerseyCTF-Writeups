# Uranus Bastion — JerseyCTF Writeup

**Challenge:** Uranus Bastion  
**Category:** misc  
**Difficulty:** Hard  
**Points:** 993  
**Solves:** 24  
**Author:** Trent  
**Flag:** `jctf{the_lattice_trusts_the_surface_not_the_soul}`

---

## Description

> The Uranus Orbital Defense Lattice guards the next hop inward. A decommissioned staging node was recovered after a failed maintenance sync, and its remnants may describe the one shipment profile the lattice still accepts. Reconstruct the protocol, rebuild the accepted coating sample, and slip your payload through the barrier.

**Attachments:** `07-uranus-bastion.zip`  
**Target:** `http://uranus-bastion.aws.jerseyctf.com:8080/`

---

## Background Knowledge (Read This First!)

### What is a Maintenance Log?

A **maintenance log** records timestamped events from a system during operation or sync. In CTF challenges, logs often contain hidden clues — such as accepted network sectors, origin port pins, and revoked trust caches — that tell you exactly how to craft a valid request to a gated system.

### What is a Hex Fragment?

**Hex encoding** converts raw bytes into a string of hexadecimal characters (0–9, a–f). In this challenge, the payload is split across three `.hex` files named by phase number. Each file decodes to a plain-text key-value configuration block.

To decode hex in Python:
```python
bytes.fromhex("48656c6c6f").decode()  # → "Hello"
```

### What is SHA-256?

**SHA-256** is a cryptographic hash function. It takes any input and produces a fixed 64-character hex digest. In this challenge, the `transit_manifest.txt` provides an `EXPECTED_SHA256` value — you must stitch the fragments in the correct order until your SHA-256 matches. If it matches, your payload is correct.

### What are Custom HTTP Headers?

HTTP requests can carry extra **headers** beyond the standard ones. Servers use custom headers (e.g. `X-Coating-Class`, `X-Origin-Port`) to enforce access control. If a header is missing or wrong, the request is silently rejected. In this challenge the defense lattice only accepts requests that match a very specific header profile recovered from the zip artifacts.

---

## Files Inside the Zip

```
07-uranus-bastion.zip
├── README.txt                  ← Overview of the challenge scenario
├── transit_manifest.txt        ← COATING_CLASS, FRAGMENT_ENCODING, EXPECTED_SHA256
├── maintenance_window.log      ← Timestamped log with port, sector, and trust info
├── sync_probe.bin              ← Binary file with embedded header names and values
└── payload_fragments/
    ├── phase_03.hex            ← Fragment 1 (hex encoded)
    ├── phase_11.hex            ← Fragment 2 (hex encoded)
    └── phase_17.hex            ← Fragment 3 (hex encoded)
```

---

## Solution — Step by Step

### Step 1 — Visit the Target

Opening the URL returns a JSON response:

```
┌──(zham㉿kali)-[~]
└─$ curl http://uranus-bastion.aws.jerseyctf.com:8080/
{"endpoint":"/upload","method":"POST","notice":"Inspection failures are not explained to remote operators.","status":"ACTIVE","system":"URANUS ORBITAL DEFENSE LATTICE"}
```

This tells us the gate accepts `POST /upload` requests. The notice warns that rejection reasons are hidden — we must get every detail exactly right.

---

### Step 2 — Extract and Read the Files

```
┌──(zham㉿kali)-[~]
└─$ unzip 07-uranus-bastion.zip -d uranus
Archive:  07-uranus-bastion.zip
  inflating: uranus/payload_fragments/phase_03.hex
  inflating: uranus/payload_fragments/phase_11.hex
  inflating: uranus/payload_fragments/phase_17.hex
  inflating: uranus/maintenance_window.log
  inflating: uranus/README.txt
  inflating: uranus/sync_probe.bin
  inflating: uranus/transit_manifest.txt

┌──(zham㉿kali)-[~]
└─$ cat uranus/transit_manifest.txt
SYNC_FAMILY=UranusSync
COATING_CLASS=ALPHA
FRAGMENT_ENCODING=hex
EXPECTED_SHA256=42aa6f011ec28d2198f81407ea91217897c712ca214ef859b068b91623d31abe
```

`transit_manifest.txt` gives us the coating class and the expected SHA-256 of the reconstructed payload.

---

### Step 3 — Read the Maintenance Log

```
┌──(zham㉿kali)-[~]
└─$ cat uranus/maintenance_window.log
[206.07.14 09:14:22] lattice ingress profile: MAINT-ORB-07
[206.07.14 09:14:22] remote rejection detail disabled for non-local operators
[206.07.14 09:14:22] accepted endpoint: POST /upload
[206.07.14 09:14:22] declared maintenance origin pin: 42107
[206.07.14 09:14:22] staging beta signature trust cache revoked after checksum drift
[206.07.14 09:14:22] fallback maintenance sector 10.10.41.0/24 revoked after relay drift
[206.07.14 09:14:22] forwarded maintenance sector: 10.10.42.0/24
[206.07.14 09:14:22] archival surface parser remains in plain-text mode
[206.07.14 09:14:22] stitch relay captures in ascending phase order
```

Key deductions:

| Log Entry | Meaning |
|-----------|---------|
| `maintenance origin pin: 42107` | `X-Origin-Port: 42107` |
| `10.10.41.0/24 revoked` | Do NOT use this subnet |
| `10.10.42.0/24 forwarded` | Use `X-Forwarded-For: 10.10.42.1` |
| `beta signature trust cache revoked` | Use `UranusSync/2.3` not `2.4-beta` |
| `stitch relay captures in ascending phase order` | Combine phase_03 → phase_11 → phase_17 |

---

### Step 4 — Inspect sync_probe.bin

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
data = open('uranus/sync_probe.bin','rb').read()
for s in data.split(b'\x00'):
    if len(s) > 4 and all(32 <= b < 127 for b in s):
        print(s.decode())
"
URSYNC_PROBE_V3
/upload
User-Agent
UranusSync/2.4-beta
UranusSync/2.3
X-Forwarded-For
X-Origin-Port
X-Coating-Class
X-Filename
coating_layer_alpha.dat
Content-Type
text/plain
```

The binary embeds all the custom header names and values as plain strings. Combined with the log (beta revoked), the correct `User-Agent` is `UranusSync/2.3`.

---

### Step 5 — Decode and Verify the Fragments

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
import hashlib
frags = [
    '434f4154494e475f4c415945525f414c5048410a5245464c45435449564954595f494e4445583d302e393832310a',
    '544845524d414c5f44454341593d535441424c450a434f52445f4d41503d55522d5341542d414c49474e2d30370a',
    '50484153455f4f46465345543d30303033414632430a535552464143455f434c4153533d415243484956414c5f53594e430a'
]
payload = b''.join(bytes.fromhex(f) for f in frags)
print(payload.decode())
print('SHA256:', hashlib.sha256(payload).hexdigest())
print('Match:', hashlib.sha256(payload).hexdigest() == '42aa6f011ec28d2198f81407ea91217897c712ca214ef859b068b91623d31abe')
"
COATING_LAYER_ALPHA
REFLECTIVITY_INDEX=0.9821
THERMAL_DECAY=STABLE
CORD_MAP=UR-SAT-ALIGN-07
PHASE_OFFSET=0003AF2C
SURFACE_CLASS=ARCHIVAL_SYNC

SHA256: 42aa6f011ec28d2198f81407ea91217897c712ca214ef859b068b91623d31abe
Match: True
```

SHA-256 matches the expected value from `transit_manifest.txt`. ✅

---

### Step 6 — Build and Run the Solve Script

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

Paste this inside:

```python
import urllib.request, zipfile

frags = [
    '434f4154494e475f4c415945525f414c5048410a5245464c45435449564954595f494e4445583d302e393832310a',
    '544845524d414c5f44454341593d535441424c450a434f52445f4d41503d55522d5341542d414c49474e2d30370a',
    '50484153455f4f46465345543d30303033414632430a535552464143455f434c4153533d415243484956414c5f53594e430a'
]

payload = b''.join(bytes.fromhex(f) for f in frags)

url = 'http://uranus-bastion.aws.jerseyctf.com:8080/upload'
req = urllib.request.Request(url, data=payload, method='POST')
req.add_header('Content-Type', 'text/plain')
req.add_header('User-Agent', 'UranusSync/2.3')
req.add_header('X-Forwarded-For', '10.10.42.1')
req.add_header('X-Origin-Port', '42107')
req.add_header('X-Coating-Class', 'ALPHA')
req.add_header('X-Filename', 'coating_layer_alpha.dat')

with urllib.request.urlopen(req) as resp:
    raw = resp.read()

with open('response.zip', 'wb') as f:
    f.write(raw)

print("Saved response.zip, extracting...")

with zipfile.ZipFile('response.zip') as z:
    for name in z.namelist():
        print(f"\n=== {name} ===")
        print(z.read(name).decode('utf-8', errors='replace'))
```

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
Saved response.zip, extracting...

=== URANUS_GATE_NOTICE.txt ===
URANUS ORBITAL DEFENSE LATTICE
LEGACY TRANSIT AUTHORIZATION
Surface profile accepted.
Material classification matches archived maintenance standards.
No threat indicators detected.
No deviation from expected exterior behavior observed.
Payload integrity verified.
Internal contents were not evaluated.
Gate alignment sequence completed.
Transit window opened under legacy synchronization rules.
This authorization was granted based on conformity.
Not inspection.
Not intent.
Proceed.

=== TRANSIT_CORDS_SATURN.txt ===
TRANSIT CORD MAPPING
URANUS → SATURN RELAY
Cord Set: UR-SAT-ALIGN-07
Phase Offset: 0003AF2C
Synchronization Mode: PASSIVE
Transit beyond Uranus is now possible.

=== MATERIAL_VERIFICATION.txt ===
MATERIAL VERIFICATION REPORT
Composite coating fragment received intact.
No structural variation detected during transmission.
Verification complete.

=== FLAG.txt ===
jctf{the_lattice_trusts_the_surface_not_the_soul}
```

---

## Full Header Map

| HTTP Header | Value | Source |
|-------------|-------|--------|
| Endpoint | `POST /upload` | JSON response + `sync_probe.bin` |
| `Content-Type` | `text/plain` | `sync_probe.bin` |
| `User-Agent` | `UranusSync/2.3` | Log (beta revoked) + `sync_probe.bin` |
| `X-Forwarded-For` | `10.10.42.1` | Log (10.10.41.0/24 revoked, 10.10.42.0/24 forwarded) |
| `X-Origin-Port` | `42107` | Log (maintenance origin pin) |
| `X-Coating-Class` | `ALPHA` | `transit_manifest.txt` |
| `X-Filename` | `coating_layer_alpha.dat` | `sync_probe.bin` |

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `unzip` | Extract the challenge zip archive | ⭐ Easy |
| `cat` | Read the log, manifest, and hex fragment files | ⭐ Easy |
| Python `bytes.fromhex()` | Decode hex fragments into the plaintext payload | ⭐ Easy |
| `hashlib.sha256` | Verify payload integrity against the expected hash | ⭐ Easy |
| `urllib.request` | Craft and send the custom HTTP POST request | ⭐⭐ Medium |
| `zipfile` | Unpack the ZIP response to read the flag | ⭐ Easy |

---

## Key Takeaways

- **Maintenance logs are goldmines** — every line of `maintenance_window.log` was a direct clue for a required header value. Always read logs carefully in forensics/misc challenges.
- **Binary files hide readable strings** — running Python string parsing on `sync_probe.bin` revealed all custom header names without needing to reverse engineer anything.
- **Order matters** — the log said "stitch relay captures in ascending phase order." Stitching in the wrong order produces a different SHA-256 and gets rejected silently.
- **Revocation clues are traps** — the log explicitly mentioned that beta signatures and the `10.10.41.0/24` sector were revoked. The challenge was designed to mislead you into using those wrong values.
- **The flag is the theme** — `jctf{the_lattice_trusts_the_surface_not_the_soul}` perfectly captures the challenge: the defense system only checked the outer coating (headers + format), not what was actually inside.
