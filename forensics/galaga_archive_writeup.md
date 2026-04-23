# Galaga Archive — JerseyCTF Writeup

**Challenge:** Galaga Archive  
**CTF:** JerseyCTF  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 983  
**Solves:** 36  
**Challenge Author:** cyb3r_s1lva  
**Writeup by:** zham  
**Flag:** `jctf{roasted_galatic_invaders}`

---

## Description

> I heard rumors that there has been work on a second sequel to the original 1980's Galaga game! I was able to listen to some activity on their network, but it looks like I needed some authentication to reach the shared archive.

**Attachment:** `galaga_galaxy_invaders2.pcap`

---

## Background Knowledge (Read This First!)

### What is a PCAP file?

A **PCAP** (Packet Capture) file is a recording of network traffic. Tools like Wireshark or `tshark` can read it to see every packet sent and received — including usernames, file transfers, and authentication attempts.

### What is Kerberos?

**Kerberos** is an authentication protocol used in Windows Active Directory environments. When a user logs in, their client sends an **AS-REQ** (Authentication Service Request) to the Domain Controller. This request contains a pre-authentication timestamp encrypted with a key derived from the user's password.

If an attacker captures this AS-REQ packet, they can extract a hash and attempt to crack it offline — this is called **Kerberos Pre-Auth cracking** (hashcat mode 7500).

### What is SMB?

**SMB** (Server Message Block) is a Windows file-sharing protocol. When a user connects to a network share like `\\DC-01\GalagaTwo`, their computer sends SMB packets to read files. If those packets are captured in a PCAP, the files can be extracted and saved to disk.

### What is XOR encryption?

**XOR** is a simple cipher. If you XOR plaintext with a key, you get ciphertext. XOR the ciphertext with the same key again and you get the plaintext back. In this challenge, the key is `SHA256(password)` — a 32-byte hash of the user's password, repeated across the full message.

### What is NTLM?

**NTLM** is a Windows password hashing format: `NTLM = MD4(UTF-16LE(password))`. It is used internally by Kerberos RC4-HMAC encryption, so cracking the Kerberos hash requires computing the NTLM hash of each candidate password.

---

## Solution — Step by Step

### Step 1 — Open the PCAP and identify key traffic

Load the PCAP with `tshark` and look at the first 50 packets:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r galaga_galaxy_invaders2.pcap 2>/dev/null | head -50
    1   0.000000 PCSSystemtec_6c:3b:4e → Broadcast    ARP 60 Who has 192.168.0.1? Tell 192.168.0.2
    8  55.762774  192.168.0.3 → 192.168.0.2  TCP 66 64600 → 445 [SYN, ECE, CWR]
   11  55.764222  192.168.0.3 → 192.168.0.2  SMB 127 Negotiate Protocol Request
   18  55.771109  192.168.0.3 → 192.168.0.2  KRB5 296 AS-REQ
   20  55.777473  192.168.0.2 → 192.168.0.3  KRB5 293 AS-REP
```

Key observations:
- **Domain:** `galaga.galaxy.org` | **DC:** `DC-01` (192.168.0.2)
- **Client:** 192.168.0.3
- **SMB2 share accessed:** `\\DC-01\GalagaTwo`
- **Files read:** `galatic_galaga_sequel.txt`, `ideas1.txt`, `ideas2.txt`, `Shareholder_Meeting_Linkedin_POST.txt`
- **Kerberos AS-REQ packets** present for users `bob` and `galatic`

---

### Step 2 — Extract the SMB files from the PCAP

Use tshark to export all SMB file objects directly to disk:

```
┌──(zham㉿kali)-[~]
└─$ mkdir /tmp/smb_export

┌──(zham㉿kali)-[~]
└─$ tshark -r /media/sf_downloads/galaga_galaxy_invaders2.pcap --export-objects smb,/tmp/smb_export/

┌──(zham㉿kali)-[~]
└─$ ls /tmp/smb_export/
%5cgalatic_galaga_sequel.txt  %5cideas1.txt  %5cideas2.txt  %5cShareholder_Meeting_Linkedin_POST.txt
```

The exported files are stored as hex text. Reading `galatic_galaga_sequel.txt` reveals the encoding scheme:

> *"to make sure no peeping eyes are watching, I used the password for the tech developer account plus an additional measure to encode the rest of the message! repeating XOR of the plaintext with SHA256(password)"*

And `Shareholder_Meeting_Linkedin_POST.txt` (marked DON'T DELETE at the top) is an embarrassing LinkedIn post draft — a red herring confirming `galatic` is the "tech developer."

The `ideas1.txt` and `ideas2.txt` files contain the XOR-encrypted ciphertext we need to decrypt.

---

### Step 3 — Extract the Kerberos RC4 hashes

The PCAP contains two Kerberos AS-REQ packets using etype 23 (RC4-HMAC). Extract the cipher values:

```
┌──(zham㉿kali)-[~]
└─$ tshark -r /media/sf_downloads/galaga_galaxy_invaders2.pcap \
  -Y "kerberos.msg_type==10 and kerberos.etype==23" \
  -T fields -e kerberos.cipher -e kerberos.CNameString 2>/dev/null
78d109e6315eb0028a09ebf5e593633150927dbeb0878c0a23a90c9d5665080cb881e39a8688b2a977f31dbcf3    bob
7bc5ac3325d1de299d13ce5350fb80d8a3f4cc9477fda57aa382cd4056f982438e5708346fb0caea55e16b8aa0    galatic
```

We also confirm from LDAP traffic that `galatic` belongs to `CN=Tech Leads` — making them the target account referenced in the sequel file.

---

### Step 4 — Crack galatic's Kerberos hash

The Kerberos RC4-HMAC pre-auth decryption works as:
- **K1** = HMAC-MD5(NTLM_key, pack('<I', 1))
- **Checksum** = first 16 bytes of cipher
- **K3** = HMAC-MD5(K1, checksum)
- **Plaintext** = RC4(K3) decrypt remaining bytes
- **Verify** = HMAC-MD5(K1, plaintext) == checksum ✓

Create the cracker:

```
┌──(zham㉿kali)-[~]
└─$ nano crack.py
```

Paste the following inside nano:

```python
from passlib.hash import nthash
import hmac, hashlib, struct
from Crypto.Cipher import ARC4

def get_ntlm(p):
    return bytes.fromhex(nthash.hash(p).replace('$NT$',''))

def check(key, cipher):
    K1 = hmac.new(key, struct.pack('<I',1), hashlib.md5).digest()
    cs = cipher[:16]
    K3 = hmac.new(K1, cs, hashlib.md5).digest()
    pt = ARC4.new(K3).decrypt(cipher[16:])
    return hmac.new(K1, pt, hashlib.md5).digest() == cs

target = bytes.fromhex('7bc5ac3325d1de299d13ce5350fb80d8a3f4cc9477fda57aa382cd4056f982438e5708346fb0caea55e16b8aa0')

print("[*] Cracking... this may take a few minutes")
with open('/usr/share/wordlists/rockyou.txt','rb') as f:
    for i, line in enumerate(f):
        pwd = line.strip()
        try:
            if check(get_ntlm(pwd.decode('utf-8','ignore')), target):
                print(f"\n[+] PASSWORD FOUND: {pwd.decode()}")
                break
        except: pass
        if i % 100000 == 0:
            print(f"  Tried {i:,} passwords...", end='\r')
```

Save and exit nano: **Ctrl+O** → **Enter** → **Ctrl+X**

```
┌──(zham㉿kali)-[~]
└─$ python3 crack.py
[*] Cracking... this may take a few minutes
  Tried 7,900,000 passwords...
[+] PASSWORD FOUND: galagalogz
```

We also crack `bob`'s hash the same way and find his password is `idk` — a red herring.

---

### Step 5 — Decrypt the flag

The two idea files together form one continuous XOR-encrypted message. The key wraps every 32 bytes (SHA256 output length), so `ideas2.txt` must be decrypted starting at byte offset `len(ideas1_ciphertext)` — not from zero.

```
┌──(zham㉿kali)-[~]
└─$ nano decrypt_combined.py
```

Paste the following inside nano:

```python
import hashlib

password = b"galagalogz"
key = hashlib.sha256(password).digest()

with open("/tmp/smb_export/%5cideas1.txt", "rb") as f:
    hex1 = f.read().decode("utf-8", errors="ignore").strip()

with open("/tmp/smb_export/%5cideas2.txt", "rb") as f:
    hex2 = f.read().decode("utf-8", errors="ignore").strip()

ct1 = bytes.fromhex(hex1)
ct2 = bytes.fromhex(hex2)

pt1 = bytes([ct1[i] ^ key[i % 32] for i in range(len(ct1))])

offset = len(ct1)
pt2 = bytes([ct2[i] ^ key[(i + offset) % 32] for i in range(len(ct2))])

print((pt1 + pt2).decode('utf-8', errors='replace'))
```

Save and exit nano: **Ctrl+O** → **Enter** → **Ctrl+X**

```
┌──(zham㉿kali)-[~]
└─$ python3 decrypt_combined.py
cool! here is my update for you. I still have not developed anything yet, a lot is still in the 
brainstorming stage. I am not sure if there should be multiple boss stages, but it should 
definitely have a more dynamic progression system, it would be dope if we implement a rogue-like 
progression system. other then that, I do not have much else to update but ill make sure to keep 
you update! jctf{roasted_galatic_invaders}
```

**Flag: `jctf{roasted_galatic_invaders}`** 🎮

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `tshark` | Analyze PCAP, export SMB objects, extract Kerberos ciphers | ⭐⭐ Medium |
| `passlib` | Compute NTLM (MD4) hashes for password candidates | ⭐⭐ Medium |
| `pycryptodome` | RC4 decryption for Kerberos pre-auth verification | ⭐⭐ Medium |
| `rockyou.txt` | Wordlist for offline password cracking | ⭐ Easy |
| `hashlib` | SHA256 key derivation for XOR decryption | ⭐ Easy |
| `nano` | Write and edit Python scripts | ⭐ Easy |

---

## Key Takeaways

- **Kerberos RC4-HMAC hashes are crackable offline** — any AS-REQ captured in a PCAP can be cracked against a wordlist without touching the target network again.
- **SMB file transfers are visible in PCAPs** — if SMB traffic is not encrypted end-to-end, captured packets can be reassembled into the original files using `tshark --export-objects`.
- **XOR with a repeating key has a gotcha** — if a message is split across multiple files, the key offset must continue from where the previous file ended, not reset to zero.
- **LDAP enumeration inside PCAPs reveals user info** — group memberships, UPNs, and account details leak through unencrypted LDAP searches, helping identify the target account.
- **Don't get distracted by red herrings** — the LinkedIn draft and `bob`'s password (`idk`) were distractions; the real path was the `galatic` tech lead account and their password `galagalogz`.
