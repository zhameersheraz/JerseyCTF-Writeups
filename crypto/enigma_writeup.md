# ENigma — JerseyCTF Writeup

**Challenge:** ENigma  
**Category:** crypto  
**Difficulty:** Medium  
**Points:** 554  
**Solves:** 26  
**Author:** Robert  
**Flag:** `jctf5{ev1l_publ1c_k3ys_w1th_evil_public_d33ds}`

---

## Description

> We have discovered an old broadcast server that does not seem to fully understand how RSA works. We know that if you can trick it into broadcasting in plain text it will give up its flag but we don't know how to trick it into not encrypting its broadcasts. Thankfully its giving information about the large primes it generated and allows users to request it use a specific public key.

**Attachment:** `chal.py`  
**Service:** `nc enigma.aws.jerseyctf.com 9001`

---

## Background Knowledge (Read This First!)

### What is RSA?

**RSA** is a public-key cryptosystem where encryption uses a public key `(e, n)` and decryption uses a private key `(d, n)`. The key relationship is:

```
d ≡ e⁻¹ (mod φ(n))
```

Where `φ(n) = (p-1)(q-1)` is Euler's totient of `n = p*q`.

Encryption works like this:
```
ciphertext = plaintext^e mod n
```

### What is Euler's Theorem?

**Euler's Theorem** states that for any integer `m` coprime to `n`:

```
m^φ(n) ≡ 1 (mod n)
```

This means if you raise a number to a multiple of `φ(n)`, you get back `1 mod n`. This is the mathematical foundation that RSA is built on — and also the key to breaking this challenge.

### What is the Identity Exponent trick?

If `e = φ(n) + 1`, then:

```
m^e mod n = m^(φ(n)+1) mod n
           = m^φ(n) * m mod n
           = 1 * m mod n
           = m
```

The encrypted output equals the original plaintext — **encryption becomes a no-op**. This is sometimes called the "identity exponent" attack.

---

## Reading the Source Code

The server (`chal.py`) does the following:

1. Generates two random primes `p` and `q` in the range `(256², 2×256²)`
2. Prints `n = p*q` and reveals `φ(n) = (p-1)*(q-1)` to the user
3. Asks the user to supply a public exponent `e`
4. Validates `e` with three checks:

```python
if e == 1 or e % tot_n == 0 or math.gcd(e, tot_n) != 1:
    print("NICE TRY, I DON'T want my public key being useless")
    exit(1)
```

5. Encrypts each 4-byte chunk of `message.txt` using `pow(num, e, p*q)`
6. Checks if the encrypted output equals the original message — if so, prints the flag

The goal is to make `pow(num, e, n) == num` for every chunk.

---

## The Vulnerability

The server reveals `φ(n)` directly in the prompt. The three validation checks are:

| Check | Purpose |
|-------|---------|
| `e != 1` | Prevent trivial identity |
| `e % tot_n != 0` | Prevent `e` being a multiple of `φ(n)` |
| `gcd(e, tot_n) == 1` | Ensure `e` is coprime with `φ(n)` |

**What's missing:** There is no check for `e ≡ 1 (mod φ(n))`.

If we send `e = φ(n) + 1`, all three checks pass:

- `e = φ(n) + 1 ≠ 1` ✓  
- `e % φ(n) = 1 ≠ 0` ✓  
- `gcd(φ(n) + 1, φ(n)) = gcd(1, φ(n)) = 1` ✓

But by Euler's Theorem, encryption with this `e` returns the plaintext unchanged — the server encrypts the message with itself, the check `decode == check` passes, and the flag is printed.

---

## Solution

### Step 1 — Connect and read the totient

```
┌──(zham㉿kali)-[~]
└─$ nc enigma.aws.jerseyctf.com 9001
The totient is 10023455968
```

The server immediately reveals `φ(n)`. Since `p` and `q` are randomly regenerated each connection, we need to automate reading this value and sending `φ(n) + 1` back before the connection drops.

### Step 2 — Write the solve script

Trying to pipe the answer manually with `echo | nc` fails because netcat exits before the server finishes printing the flag. We need a Python script to keep the connection alive:

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

```python
import socket
import time

s = socket.socket()
s.connect(('enigma.aws.jerseyctf.com', 9001))

data = s.recv(4096).decode()
print('Received:', data)

tot_n = int(data.split('totient is ')[1].strip())
e = tot_n + 1
print(f'Sending e = {e}')

s.send(str(e).encode() + b'\n')

time.sleep(2)
resp = s.recv(4096).decode()
print('Response:', resp)
s.close()
```

### Step 3 — Run the script and get the flag

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
Received: The totient is 7505586400
Sending e = 7505586401
Response: E IS  7505586401
THE FLAG IS jctf5{ev1l_publ1c_k3ys_w1th_evil_public_d33ds}
```

---

## Why `echo | nc` Doesn't Work

```
┌──(zham㉿kali)-[~]
└─$ echo "10023455969" | nc enigma.aws.jerseyctf.com 9001
The totient is 14364508320E IS  10023455969
```

Two problems:
1. The totient changes every connection — the hardcoded number was already stale
2. `nc` exits as soon as stdin closes (after echo), before the server responds with the flag

The Python script solves both: it parses the live totient dynamically and holds the socket open to receive the full response.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Initial connection to explore the server | ⭐ Easy |
| `python3 socket` | Automate the connection, parse totient, send answer | ⭐⭐ Medium |
| Euler's Theorem | Mathematical insight to craft the evil `e` | ⭐⭐ Medium |

---

## Key Takeaways

- **Never reveal φ(n)** — RSA security depends on keeping the totient secret. Exposing it lets an attacker craft arbitrary exponents.
- **Input validation gaps are dangerous** — the server checked `e` is coprime with `φ(n)` but forgot to check `e ≡ 1 (mod φ(n))`, which is the exact condition that breaks encryption.
- **Euler's Theorem is a double-edged sword** — it is what makes RSA work, but it is also what allows the identity exponent attack when `e = φ(n) + 1`.
- **Keep sockets alive when automating CTF challenges** — `echo | nc` is convenient but unreliable for challenges that send a delayed or multi-part response.
