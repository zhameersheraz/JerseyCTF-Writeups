# SOS — JerseyCTF Writeup

**Challenge:** SOS  
**Category:** bin  
**Difficulty:** Easy  
**Points:** 227  
**Solves:** 41  
**Challenge Author:** Samar Yagoubi  
**Writeup by:** zham  
**Flag:** `jctf{lost_in_space}`

---

## Description

> oh noooo one of our astronauts drifted away during a moonwalk. Mission control only got one boring looking message back, but they think the real SOS is hiding somewhere inside it.
>
> Find the hidden message and bring them home.

**Attachments:** `astro_beacon`, `sos_message.txt`

---

## Background Knowledge (Read This First!)

### What is Zero-Width Character Steganography?

**Steganography** is the practice of hiding secret information inside ordinary-looking data. Unlike encryption, which scrambles data to make it unreadable, steganography hides the *existence* of the message entirely.

**Zero-width characters** are Unicode characters that have no visual width — they are completely invisible when rendered in a text editor or browser. Two commonly used ones are:

```
┌──(zham㉿kali)-[~]
└─$ printf '\u200b' | xxd
00000000: e2 80 8b                                 ...        ← ZWSP (U+200B) = binary 1

┌──(zham㉿kali)-[~]
└─$ printf '\u200c' | xxd
00000000: e2 80 8c                                 ...        ← ZWNJ (U+200C) = binary 0
```

Both characters print nothing visible — but the hex dump proves they are really there as 3-byte UTF-8 sequences. By mapping them to binary `1` and `0`, an attacker (or astronaut!) can embed a secret binary message inside perfectly normal-looking text:

```
ZWSP (U+200B)  →  e2 80 8b  →  1
ZWNJ (U+200C)  →  e2 80 8c  →  0
```

For example, the sequence `ZWSP ZWNJ ZWNJ ZWSP` encodes binary `1001`. Every 8 bits make one ASCII character.

To detect these characters in a suspicious file:

```
┌──(zham㉿kali)-[~]
└─$ xxd sos_message.txt | head -4
00000000: 6f78 7967 656e 2063 6865 636b 2073 6179  oxygen check say
00000010: 7320 6576 6572 7974 68e2 808b e280 8ce2  s everyth.......
00000020: 808c e280 8be2 808c e280 8be2 808c e280  ................
00000030: 8ce2 808b e280 8be2 808b e280 8ce2 808c  ................
```

The `e2 80 8b` and `e2 80 8c` bytes hiding between normal ASCII text are the giveaway.

### What is an ELF Binary?

**ELF** (Executable and Linkable Format) is the standard executable file format on Linux systems — equivalent to `.exe` on Windows. The `astro_beacon` file has no extension but is an ELF binary compiled for x86-64 Linux.

Before running an unknown binary, always inspect it first:
```bash
file astro_beacon          # identify file type
strings astro_beacon       # extract readable strings
chmod +x astro_beacon      # make it executable
```

`strings` is especially useful — it dumps all readable text embedded in a binary, which often reveals menu options, file names, or hidden clues.

### What is Binary-to-ASCII Conversion?

Every text character has a numeric value defined by the **ASCII** standard. For example:

| Binary | Decimal | ASCII |
|--------|---------|-------|
| 01101010 | 106 | `j` |
| 01100011 | 99 | `c` |
| 01110100 | 116 | `t` |
| 01100110 | 102 | `f` |

By grouping extracted bits into sets of 8 and converting each group, hidden binary data becomes readable text.

---

## Solution — Step by Step

### Step 1 — Inspect the files

Start by understanding what we're working with:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file sos_message.txt astro_beacon
sos_message.txt: Unicode text, UTF-8 text, with no line terminators
astro_beacon:    ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=b8b8d28e04b9fb84121db0b1c28114285c104523, for GNU/Linux 3.2.0, not stripped

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat sos_message.txt
oxygen check says everything is fine please ignore this suuuuuper boring space report
```

The message looks completely normal. But something feels off — let's look closer.

### Step 2 — Detect hidden characters in the message

A hex dump reveals invisible bytes hiding between the visible characters:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xxd sos_message.txt | head -5
00000000: 6f78 7967 656e 2063 6865 636b 2073 6179  oxygen check say
00000010: 7320 6576 6572 7974 68e2 808b e280 8ce2  s everyth.......
00000020: 808c e280 8be2 808c e280 8be2 808c e280  ................
00000030: 8be2 808b e280 8ce2 808c e280 8be2 808b  ................
00000040: e280 8be2 808c e280 8ce2 808b e280 8ce2  ................
```

The bytes `e2 80 8b` (ZWSP) and `e2 80 8c` (ZWNJ) appear repeatedly between normal text. These are our hidden binary digits.

### Step 3 — Inspect the astro_beacon binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings astro_beacon
 LITTLE ASTRONAUT SOS BOX v0.9
 last ping: somewhere past the moon
result.txt
uh oh could not open result.txt
Enter normal looking space message:
Enter secret astronaut message:
space message written to result.txt
decode_result.txt
uh oh could not open decode_result.txt
Paste the weird space message:
raw sos bits written to decode_result.txt
Pick rescue mode [e]ncode / [d]ecode:
encode_rev.c
zwnj
zwsp
main
decode
encode
```

The binary is a tool with two modes: **encode** and **decode**. The `[d]ecode` mode accepts a steganographic message and writes the raw binary bits to `decode_result.txt`. This is exactly what we need.

### Step 4 — Run the decoder

Make the binary executable and run it in decode mode, piping in the suspicious message:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x astro_beacon

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat sos_message.txt | (echo "d"; cat) | ./astro_beacon
 LITTLE ASTRONAUT SOS BOX v0.9
 last ping: somewhere past the moon
Pick rescue mode [e]ncode / [d]ecode: Paste the weird space message: raw sos bits written to decode_result.txt
```

The binary extracts the hidden bits and writes them to `decode_result.txt`.

### Step 5 — Convert the binary bits to ASCII

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat decode_result.txt
oxygen check says everyth0110101001100011011101000110011001111011011011000110111101110011011101000101ing is fine please ignore1111011010010110111001011111011100110111000001100001011000110110010101111101 this suuuuuper boring space report
```

Extract only the `0` and `1` characters and decode them. Create the script using nano:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano solve.py
```

Paste the script below into nano:

```python
import re

with open('decode_result.txt', 'r') as f:
    content = f.read()

# Extract only binary digits
bits = re.sub(r'[^01]', '', content)
print(f"Extracted bits ({len(bits)} total): {bits}")

# Decode 8 bits at a time to ASCII
decoded = ''
for i in range(0, len(bits) - 7, 8):
    byte = bits[i:i+8]
    decoded += chr(int(byte, 2))

print(f"Decoded message: {decoded}")
```

Save and exit nano:
- `Ctrl + O` → write/save the file
- `Enter` → confirm the filename (`solve.py`)
- `Ctrl + X` → exit nano

Then run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 solve.py
Extracted bits (152 total): 01101010011000110111010001100110011110110110110001101111011100110111010001011111011010010110111001011111011100110111000001100001011000110110010101111101
Decoded message: jctf{lost_in_space}
```

### Step 6 — Submit the flag

```
jctf{lost_in_space}
```

The astronaut is home. 🚀

---

## One-liner Grab Script

Here is a single reusable script that automates Steps 4 and 5.

Create the file using nano:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano solve.sh
```

Paste the script below into nano:

```bash
#!/bin/bash
chmod +x ./astro_beacon
cat sos_message.txt | (echo "d"; cat) | ./astro_beacon > /dev/null
python3 -c "
import re
with open('decode_result.txt') as f:
    bits = re.sub(r'[^01]', '', f.read())
print(''.join(chr(int(bits[i:i+8], 2)) for i in range(0, len(bits)-7, 8)))
"
```

Save and exit nano:
- `Ctrl + O` → write/save the file
- `Enter` → confirm the filename (`solve.sh`)
- `Ctrl + X` → exit nano

Then make it executable and run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x solve.sh

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ./solve.sh
jctf{lost_in_space}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Identify file types (ELF binary vs text) | ⭐ Easy |
| `strings` | Inspect readable content inside the ELF binary | ⭐ Easy |
| `xxd` | Hex dump to detect invisible Unicode characters | ⭐ Easy |
| `astro_beacon` | Decode the steganographic message into raw bits | ⭐ Easy |
| `python3` | Convert extracted binary bits to ASCII text | ⭐ Easy |

---

## Key Takeaways

- **Invisible characters are a real steganography technique** — Unicode zero-width characters (U+200B, U+200C) are completely invisible in normal text editors and browsers, making them ideal for hiding data in plain sight.
- **Always hex-dump suspicious text files** — `xxd` or `xxd | head` immediately exposes hidden non-printable bytes that `cat` will never show you.
- **`strings` is your first move on unknown binaries** — before running anything, dump its readable strings to understand what the program does and what files it reads/writes.
- **The challenge tools are often the solution** — the `astro_beacon` encoder/decoder binary was provided precisely because it is the intended decoding tool. When a binary is attached, it's usually there to be used.
- **Binary-to-ASCII is always 8 bits per character** — once you have a clean binary string, group into bytes and convert with `chr(int(byte, 2))` in Python.
