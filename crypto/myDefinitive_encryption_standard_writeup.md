# myDefinitive Encryption Standard — JerseyCTF Writeup

**Challenge:** myDefinitive Encryption Standard  
**Category:** crypto  
**Difficulty:** Medium  
**Points:** 701  
**Solves:** 55  
**Challenge Author:** cyb3r_s1lva  
**Writeup by:** zham  
**Flag:** `jctf{f3ist&l_fun_w1thout_sb0x}`

---

## Description

> We have been asked to create a similar encryption standard to DES to work on lightweight IBM computer systems. In order to test its strength against potential adversaries, can you retrieve the original plaintext in the given file with the encryption steps? The key might be missing a letter or two ...

**Attachment:** `jersey_chall_file.py`

---

## Background Knowledge (Read This First!)

### What is a Feistel Cipher?

A **Feistel cipher** is a symmetric block cipher structure used in many classical encryption schemes, most famously **DES (Data Encryption Standard)**. It works by:

1. Splitting each block of plaintext into a **left half (L)** and a **right half (R)**
2. Running a series of **rounds**, where in each round:
   - The new left becomes the old right: `L_new = R`
   - The new right becomes the old left XORed with a round function applied to the right: `R_new = L ^ F(R, key)`
3. After all rounds, swapping L and R one final time

The beautiful property of Feistel ciphers is that **encryption and decryption use the exact same structure** — you only need to reverse the order of the round keys to decrypt.

```
Encrypt:  L,R → (R, L ^ F(R,k1)) → (L^F(R,k1), R^F(...,k2)) → ...
Decrypt:  reverse key order, same operations
```

### What is the Golden Ratio Constant?

The value `0x9E3779B9` is the **32-bit representation of the golden ratio** (φ = 1.6180...) scaled to fit a 32-bit integer:

```
floor(2^32 / φ) = 0x9E3779B9
```

This constant appears in many cryptographic algorithms (TEA, XTEA, SHA-1 init) because it produces excellent mixing with no obvious mathematical backdoor.

### What is a Key Schedule?

The **key schedule** is the algorithm that derives a set of **round keys** (subkeys) from the main encryption key. In this challenge, `aaa(key)` is the key schedule — it rotates the key left by 3 bits each round, then XORs with an increasing multiple of the golden ratio constant.

### What is Brute-Forcing?

When you know a key is *almost* correct — differing by only a small number of bytes — you can **brute-force** it by trying every possible value for those unknown bytes. With a 1-byte unknown, that is only 256 attempts. Even a 2-byte search is only 65,536 attempts, trivially fast on a modern machine.

---

## Solution — Step by Step

### Step 1 — Read the challenge file

Open `jersey_chall_file.py` and inspect its contents:

```
┌──(zham㉿kali)-[~]
└─$ cat jersey_chall_file.py
ROUNDS = 4
rot = lambda x,n: ((x<<n)&0xffffffff)|(x>>(32-n))

def aaa(k):
    ks=[]
    for i in range(ROUNDS):
        k=rot(k,3)
        ks.append((k^(0x9E3779B9*(i+1)))&0xffffffff) # Golden Ratio: 0x9E3779B9
    return ks

def bbb(r,k):
    return (rot(r^k,5)*0x45D9F3B)&0xffffffff

def ccc(block,keys):
    l=int.from_bytes(block[:4],"big")
    r=int.from_bytes(block[4:],"big")
    for k in keys:
        l,r=r,l^bbb(r,k) # t = l ^ bbb(r,k)
    # final swap back
    return r.to_bytes(4,"big")+l.to_bytes(4,"big")

def encrypt(data,key):
    ks=aaa(key)
    return b''.join(ccc(data[i:i+8],ks) for i in range(0,len(data),8))

if __name__ == "__main__":
    key_hint=0xD4D4A1A0
    ciphertext=b"9\xbd/\x9588\x0bwo\xce+\xd4*\xd8\xda\x8d\x1f*\xac\x07f\xf1a\x9b\xd7$O\xbdU\\\xe2\xc5"
    print("cipher:",ciphertext.hex())
```

Two things stand out immediately:
- The structure is a **4-round Feistel cipher** — it has the tell-tale `l,r = r, l ^ F(r,k)` pattern.
- The challenge says the key might be **"missing a letter or two"**, meaning `key_hint` is not the real key, but very close to it.

Run the file to confirm the ciphertext hex:

```
┌──(zham㉿kali)-[~]
└─$ python3 jersey_chall_file.py
cipher: 39bd2f953838086b776fce2bd42ad8da8d1f2aac0766f1619bd7244fbd555ce2c5
```

### Step 2 — Understand the Feistel structure

Trace through what `ccc()` does on a single 8-byte block with keys `[k0, k1, k2, k3]`:

```
Start:   (L0, R0)
Round 1: (L1, R1) = (R0,  L0 ^ bbb(R0, k0))
Round 2: (L2, R2) = (R1,  L1 ^ bbb(R1, k1))
Round 3: (L3, R3) = (R2,  L2 ^ bbb(R2, k2))
Round 4: (L4, R4) = (R3,  L3 ^ bbb(R3, k3))
Output:  R4 || L4   ← note the final swap
```

The output is written as `r.to_bytes() + l.to_bytes()`, which is a **final swap** — a standard Feistel finishing step. This must be carefully accounted for when writing the decryption function.

### Step 3 — Write the decryption function

Because Feistel ciphers are **self-inverse**, decryption uses the same round function with keys applied in **reverse order**. We read the ciphertext output `(R4, L4)` back in and undo each round:

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

```python
ROUNDS = 4
rot = lambda x,n: ((x<<n)&0xffffffff)|(x>>(32-n))

def aaa(k):
    ks=[]
    for i in range(ROUNDS):
        k=rot(k,3)
        ks.append((k^(0x9E3779B9*(i+1)))&0xffffffff)
    return ks

def bbb(r,k):
    return (rot(r^k,5)*0x45D9F3B)&0xffffffff

def decrypt_block(block, keys):
    r = int.from_bytes(block[:4], "big")   # this is R4 (first 4 bytes of output)
    l = int.from_bytes(block[4:], "big")   # this is L4 (last 4 bytes of output)
    for k in reversed(keys):              # reverse key order to undo rounds
        l, r = r ^ bbb(l, k), l
    return l.to_bytes(4,"big") + r.to_bytes(4,"big")

def decrypt(data, key):
    ks = aaa(key)
    return b''.join(decrypt_block(data[i:i+8], ks) for i in range(0, len(data), 8))
```

### Step 4 — Brute-force the missing key bytes

The hint says the key is `0xD4D4A1A0` but might differ by "a letter or two." We try changing **one byte at a time** across all four byte positions — 256 × 4 = **1,024 attempts total**, which completes in under a second:

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[*] Trying 1-byte differences from key hint 0xD4D4A1A0 ...
```

The brute-force loop added to `solve.py`:

```python
ciphertext = b"9\xbd/\x9588\x0bwo\xce+\xd4*\xd8\xda\x8d\x1f*\xac\x07f\xf1a\x9b\xd7$O\xbdU\\\xe2\xc5"
key_hint   = 0xD4D4A1A0

print(f"[*] Trying 1-byte differences from key hint 0x{key_hint:08X} ...")
for pos in range(4):
    for val in range(256):
        key = (key_hint & ~(0xFF << (8*pos))) | (val << (8*pos))
        pt  = decrypt(ciphertext, key)
        if b'jctf{' in pt:
            print(f"[!] KEY FOUND: 0x{key:08X}")
            print(f"    Plaintext: {pt}")
```

### Step 5 — Read the flag

The brute-force immediately produces a hit:

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[*] Trying 1-byte differences from key hint 0xD4D4A1A0 ...
[!] KEY FOUND: 0xD4D4A1A5
    Plaintext: b'jctf{f3ist&l_fun_w1thout_sb0\x00\x00x}'
```

The null bytes `\x00\x00` are padding from block-aligned encryption. Strip them to get the clean flag:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
pt = b'jctf{f3ist&l_fun_w1thout_sb0\x00\x00x}'
print(pt.replace(b'\x00', b'').decode())
"
jctf{f3ist&l_fun_w1thout_sb0x}
```

The key differed from the hint by only the **last byte**: `A0 → A5` — exactly "a letter or two" as the challenge promised.

---

## Full Solve Script

```python
ROUNDS = 4
rot = lambda x,n: ((x<<n)&0xffffffff)|(x>>(32-n))

def aaa(k):
    ks=[]
    for i in range(ROUNDS):
        k=rot(k,3)
        ks.append((k^(0x9E3779B9*(i+1)))&0xffffffff)
    return ks

def bbb(r,k):
    return (rot(r^k,5)*0x45D9F3B)&0xffffffff

def decrypt_block(block, keys):
    r = int.from_bytes(block[:4], "big")
    l = int.from_bytes(block[4:], "big")
    for k in reversed(keys):
        l, r = r ^ bbb(l, k), l
    return l.to_bytes(4,"big") + r.to_bytes(4,"big")

def decrypt(data, key):
    ks = aaa(key)
    return b''.join(decrypt_block(data[i:i+8], ks) for i in range(0, len(data), 8))

ciphertext = b"9\xbd/\x9588\x0bwo\xce+\xd4*\xd8\xda\x8d\x1f*\xac\x07f\xf1a\x9b\xd7$O\xbdU\\\xe2\xc5"
key_hint   = 0xD4D4A1A0

print(f"[*] Trying 1-byte differences from key hint 0x{key_hint:08X} ...")
for pos in range(4):
    for val in range(256):
        key = (key_hint & ~(0xFF << (8*pos))) | (val << (8*pos))
        pt  = decrypt(ciphertext, key)
        if b'jctf{' in pt:
            flag = pt.replace(b'\x00', b'').decode('ascii', errors='replace')
            print(f"[!] KEY FOUND: 0x{key:08X}  (byte {pos} changed: 0x{(key_hint>>(8*pos))&0xff:02X} -> 0x{val:02X})")
            print(f"    Flag: {flag}")
```

Run it:

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[*] Trying 1-byte differences from key hint 0xD4D4A1A0 ...
[!] KEY FOUND: 0xD4D4A1A5  (byte 0 changed: 0xA0 -> 0xA5)
    Flag: jctf{f3ist&l_fun_w1thout_sb0x}
```

---

## Flag Decoded

The flag is leet-speak that describes exactly what the cipher is:

| Part | Leet-speak | Meaning |
|------|-----------|---------|
| `f3ist&l` | `3`=e, `&`=e | **Feistel** |
| `fun` | — | fun |
| `w1thout` | `1`=i | **without** |
| `sb0x` | `0`=o | **S-box** |

**"Feistel fun without S-box"** — a perfect self-description of `myDES`: it copies the Feistel structure from DES but omits the S-boxes entirely, which are what actually provide non-linearity and real cryptographic strength in real DES.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `cat` | Read and inspect the challenge Python file | ⭐ Easy |
| `python3` | Run the challenge script to confirm the ciphertext | ⭐ Easy |
| Feistel reversal | Decrypt by reversing key order — no special library needed | ⭐⭐ Medium |
| Brute-force loop | Try 1,024 key candidates (4 positions × 256 values) | ⭐ Easy |

---

## Key Takeaways

- **Feistel ciphers are their own inverse** — if you can encrypt, you can decrypt with no extra work. Just reverse the round key order and apply the same logic.
- **"Missing a letter or two" means brute-force** — when a challenge hints the key is *almost* known, always try single-byte variations first. 1,024 attempts is essentially instant.
- **The final swap matters** — the output of `ccc()` is `r || l` (swapped), not `l || r`. Forgetting this produces garbage output even with the correct key and correct round logic.
- **S-boxes are not optional in real DES** — their absence here means the cipher has no non-linearity: every operation is just XOR, rotation, and multiplication. Real DES derives much of its strength from the S-box substitution step.
- **Read the flag, not just the output** — the flag itself told you exactly what the cipher was (`f3ist&l_fun_w1thout_sb0x`), a useful sanity-check that your decryption is correct.
