# Shall We Play a Game? — JerseyCTF Writeup

**Challenge:** Shall We Play a Game?  
**Category:** misc  
**Difficulty:** Hard  
**Points:** 908  
**Solves:** 81  
**Challenge Author:** Garrett Gonzalez-Rivas  
**Writeup by:** zham  
**Flag:** `jctf{6r3371N65_Pr0F3550r_F41K3N}`

---

## Description

> A suspicious arcade game was left by the original developer on the system. It runs a simple tic-tac-toe game, but something feels off...
>
> You were able to copy the program and its assets onto your system, dig into the binary, figure out what it's really doing, and flip the right switch before the game ends.

**NOTE:** the challenge is best experienced on a terminal which implements the Kitty Graphics Protocol.

**Attachments:** `tictactoe`, `board.png`

---

## Hints

- **Hint 1:** The developer left notes about something called "SPRT" for rendering the image, and there aren't any other images for rendering the actual X's and O's.
- **Hint 2:** The PNG file is larger than a typical board image. What lives after the end of a PNG?
- **Hint 3:** Search for a magic number — the struct you find has two sets of chunk descriptors... what are they there for?

---

## Background Knowledge (Read This First!)

### What is a PNG Trailer?

A valid PNG file ends with an **IEND chunk** — a fixed 12-byte sequence that signals the end of the image. Any data written **after** the IEND chunk is ignored by image viewers, but it is still present in the raw file bytes. This makes PNG files a popular hiding spot for extra data in steganography and CTF challenges.

To find where IEND is located:

```python
data = open('board.png', 'rb').read()
iend = data.rfind(b'IEND')
trailer = data[iend + 8:]   # skip 4-byte CRC after IEND
```

You can verify a PNG has hidden trailer data using `exiftool`:

```
exiftool board.png
# Warning: [minor] Trailer data after PNG IEND chunk [x189]
```

The `[x189]` warning tells us there are 189 bytes of non-PNG data appended after the image ends — the hidden payload.

### What is XOR Encoding?

**XOR** (exclusive OR) is a simple bitwise operation. In shellcode and malware, data is often XOR-encoded to hide strings from basic string searches. Each byte is XOR'd against a fixed key byte. To decode it, you XOR each byte again with the same key.

For example, if the key is `0x80`:
```
encoded byte: 0xF0
0xF0 ^ 0x80 = 0x70  →  'p'
```

In this challenge, the payload embedded in the PNG trailer contains XOR-encoded strings with key `0x80`. The strings are invisible to `strings` on the raw file, but become readable after decoding.

### What is Shellcode?

**Shellcode** is a sequence of machine code instructions (raw bytes) embedded in a file or memory. In CTF reversing challenges, shellcode is often hidden inside other files and is meant to be executed at runtime. Common patterns include:

- `mov rcx, N` — set a loop counter
- `xor [rdi], 0x80` — XOR decode a byte
- `syscall` — make a system call
- `lea rdi, [rip + offset]` — load an address relative to the current instruction pointer

Recognizing these byte patterns lets you find and decode shellcode without running it.

### What is `exiftool`?

**exiftool** is a command-line tool that reads metadata embedded inside files. It supports hundreds of formats including PNG, JPEG, ELF, PDF, and more. It is useful for quickly scanning a file for hidden warnings or unusual metadata fields.

### What is `strings`?

**strings** is a command-line utility that extracts sequences of printable ASCII characters from any binary file. It is often the first tool used when reversing an unknown binary — it can reveal hardcoded messages, paths, library names, and encoded data.

---

## Solution — Step by Step

### Step 1 — Inspect the files with exiftool

Start by running `exiftool` on both files to understand what you're dealing with:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool tictactoe
File Type                       : ELF shared library
CPU Architecture                : 64 bit
CPU Type                        : AMD x86-64
Object File Type                : Shared object file

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool board.png
File Type                       : PNG
Image Width                     : 480
Image Height                    : 480
Warning                         : [minor] Trailer data after PNG IEND chunk [x189]
```

The critical finding: `board.png` has **trailer data after the IEND chunk**. The image is just a normal 480×480 tic-tac-toe board, but 34,454 bytes of hidden data are appended after it ends.

### Step 2 — Extract and inspect the PNG trailer

Locate the IEND chunk and dump everything after it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
data = open('board.png','rb').read()
iend = data.rfind(b'IEND')
print(f'IEND at offset: {iend}')
print(f'Trailer starts at: {iend+8}')
trailer = data[iend+8:]
print(f'Trailer length: {len(trailer)} bytes')
"
IEND at offset: 1855
Trailer starts at: 1863
Trailer length: 34454 bytes
```

The trailer is 34,454 bytes — far too large to be coincidental padding. Scanning the hex dump of the trailer reveals two distinct regions: a large block of mostly null bytes with scattered pixel-like values (`191c29ff`), followed by a dense block of non-null bytes at the end that looks like executable code.

### Step 3 — Search for known strings in the binary

Run `strings` on the game binary to look for known function names and keywords:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings tictactoe | grep -i "sprt\|sprite\|jctf\|flag"
SPRT
load_sprite
sprite_config
```

The string `SPRT` matches **Hint 1** — the developer left notes about a custom sprite format for rendering X's and O's. The binary references a `load_sprite` function and a `sprite_config` struct. No flag is visible in plain text, which means it is encoded.

### Step 4 — Locate XOR-encoded shellcode in the trailer

The trailer ends with what looks like shellcode. Search for x86-64 XOR loop patterns — specifically `mov rcx, N` (the loop counter opcode `\x48\xc7\xc1`) followed by `xor [rdi], 0x80; loop` (`\x80\x37\x80`):

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
data = open('board.png','rb').read()
iend = data.rfind(b'IEND')
trailer = data[iend+8:]
pattern = b'\x48\xc7\xc1'
for i in range(len(trailer)-50):
    if trailer[i:i+3] == pattern and trailer[i+7:i+10] == b'\x80\x37\x80':
        count = trailer[i+3]
        lea_pos = i - 7
        if trailer[lea_pos:lea_pos+3] == b'\x48\x8d\x3d':
            rip_off = int.from_bytes(trailer[lea_pos+3:lea_pos+7], 'little')
            target = lea_pos + 7 + rip_off
            dec = bytes(b ^ 0x80 for b in trailer[target:target+count])
            print(f'Decrypted at {target} ({count} bytes): {dec}')
"
Decrypted at 32889 (70 bytes): b'R~\x8d...'
Decrypted at 33076 (39 bytes): b'g\xfe\xb8...'
```

Two XOR-encoded blobs are found. The raw XOR with `0x80` alone doesn't reveal readable strings — the key might be different, or the blobs use a secondary encoding. Move on to the brute-force approach.

### Step 5 — Brute-force decode the full trailer with XOR key 0x80

XOR the **entire** trailer with `0x80` and scan for printable ASCII strings of 8 or more characters:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
data = open('board.png','rb').read()
iend = data.rfind(b'IEND')
trailer = data[iend+8:]
xored = bytes(b ^ 0x80 for b in trailer)
import re
for m in re.finditer(b'[\x20-\x7e]{8,}', xored):
    print(m.start(), m.group().decode())
"
33274 [!] Oh4>
33572 1+Kx_x\R
33716 p= no, you've been pwned!
33874 [*] jctf{6r3371
34043 is system has been compromised.
34268 9N65_Pr0F3550r_F41K3N}
```

The flag is split across two offsets in the decoded output:

- Offset **33874**: `[*] jctf{6r3371`
- Offset **34268**: `N65_Pr0F3550r_F41K3N}`

Joining them together gives the complete flag.

---

## Flag

```
jctf{6r3371N65_Pr0F3550r_F41K3N}
```

The flag is a reference to the film **WarGames (1983)**, where the AI character is named **Professor Falken** and famously greets the player with *"Greetings, Professor Falken"* — fitting perfectly for a challenge titled "Shall We Play a Game?"

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `exiftool` | Detect hidden trailer data after PNG IEND chunk | ⭐ Easy |
| `strings` | Find readable strings and SPRT keyword in the binary | ⭐ Easy |
| `python3` | Extract PNG trailer, locate shellcode patterns, XOR decode | ⭐⭐ Medium |
| `re` (Python) | Scan XOR-decoded bytes for printable ASCII flag strings | ⭐ Easy |

---

## Key Takeaways

- **Always check PNG files for trailer data** — `exiftool` will warn you with `Trailer data after PNG IEND chunk` if extra bytes are appended after the image ends.
- **XOR encoding hides strings from `strings`** — a simple `strings` scan on the PNG would show nothing. XOR-decoding the trailer with key `0x80` immediately reveals the flag.
- **Shellcode patterns are recognizable** — even without a disassembler, you can spot x86-64 XOR decode loops by looking for the opcode sequence `\x48\xc7\xc1` (mov rcx) followed by `\x80\x37\x80` (xor [rdi]).
- **Split flags are common** — the flag was split across two separate offsets in the decoded data. Always concatenate all readable strings found and look for `jctf{` to identify the start.
- **Pop culture references make great flag text** — recognizing *WarGames* would have hinted at the content of the flag even before fully decoding it.
