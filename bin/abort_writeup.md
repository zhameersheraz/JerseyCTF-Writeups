# Abort — JerseyCTF Writeup

**Challenge:** Abort  
**Category:** bin  
**Difficulty:** Medium  
**Points:** 598  
**Solves:** 96  
**Author:** Samar Yagoubi  
**Writeup by:** zham  
**Flag:** `jctf{$UccES5Fully_abOrt3D!_cOnGRATUl@t!0ns}`

---

## Description

> The arcade cabinet's control node is failing, and maintenance access has been restricted. You've intercepted the executable powering the system, but only the compiled binary remains.
>
> Reverse engineer the program, discover what input it expects, and restore the cabinet before the watchdog timer powers everything down.
>
> **Note:** You will have limited time once connected.

**Connection:** `nc abort.aws.jerseyctf.com 1337`  
**Attachment:** `abort` (ELF 64-bit binary)

---

## Background Knowledge (Read This First!)

### What is a compiled binary / ELF?

An **ELF** (Executable and Linkable Format) is the standard binary format on Linux. When a C program is compiled, the source code is gone — only machine code remains. To understand what the program does, we use **reverse engineering**: reading the disassembled instructions and working backwards to figure out the original logic.

### What is disassembly?

**Disassembly** converts raw machine code back into human-readable assembly instructions. Tools like `objdump` and Ghidra do this automatically. Even without source code, we can read the assembly to understand what checks the program performs on our input.

Key x86-64 assembly instructions used in this challenge:

| Instruction | Meaning |
|-------------|---------|
| `xor eax, VALUE` | Compute `eax = eax XOR VALUE` |
| `sub eax, VALUE` | Compute `eax = eax - VALUE` |
| `cmp eax, VALUE` | Compare `eax` to `VALUE` (sets flags) |
| `je ADDR` | Jump if equal (previous `cmp` was zero) |
| `movzbl (%rax), eax` | Load 1 byte from memory at `rax` into `eax` |

### What is a watchdog timer?

A **watchdog timer** is a safety mechanism that automatically shuts down a system if it does not receive a correct signal within a time limit. In this challenge, the binary calls `alarm(0x78)` (120 seconds) at startup — if the correct input is not sent within that window, the connection closes with `[ SESSION TIMEOUT ]`.

### What is a raw binary payload?

Programs that communicate over `nc` (netcat) read raw bytes from stdin. When we need to send non-printable bytes (like `\x0e`, `\xca`, `\x83`), we cannot type them by hand. We use **Python** to build the exact byte sequence and pipe it directly to `nc`.

---

## Reconnaissance

### Step 1 — Identify the binary

```
┌──(zham㉿kali)-[~]
└─$ file abort
abort: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, stripped
```

It is a **stripped** binary (no debug symbols), so function names are gone. We will rely on disassembly and `strings` to understand it.

### Step 2 — Extract readable strings

```
┌──(zham㉿kali)-[~]
└─$ strings abort
...
cat flagH
lag.txt
[ SESSION TIMEOUT ]
[ CABINET POWERED DOWN ]
[ SYSTEM OVERRIDE ACCEPTED ]
[ ACCESS LEVEL: ARCADE OPS ]
[ DUMPING VAULT CONTENTS ]
NEON ARCADE CONTROL NODE v1.3.3
BOOT MESSAGE: Repair the cabinet link.
STATUS: Signal drift detected.
Insert maintenance packet:
[ LINK FAILURE ]
[ CABINET REMAINS LOCKED ]
...
```

Two things stand out immediately:
- The binary runs `cat flag.txt` on success — that is how the flag is read.
- There are both success (`[ SYSTEM OVERRIDE ACCEPTED ]`) and failure (`[ LINK FAILURE ]`) messages, confirming this is a classic **input validation** challenge.

### Step 3 — Connect and observe the behaviour

```
┌──(zham㉿kali)-[~]
└─$ nc abort.aws.jerseyctf.com 1337
  NEON ARCADE CONTROL NODE v1.3.3
BOOT MESSAGE: Repair the cabinet link.
STATUS: Signal drift detected.
Insert maintenance packet:
[ SESSION TIMEOUT ]
[ CABINET POWERED DOWN ]
```

The program reads input and then immediately times out if the input is wrong or takes too long. We need to send the correct 80-byte packet **all at once** (piped, not typed).

---

## Reverse Engineering

### Step 4 — Disassemble the binary

```
┌──(zham㉿kali)-[~]
└─$ objdump -d abort > abort.asm
```

The main function sets an `alarm(0x78)` watchdog, then calls the challenge function which performs three separate checks on an 80-byte input buffer.

### Understanding the three checks

The binary reads **80 bytes** from stdin into a global buffer, then checks three specific regions of that buffer:

**Check 1 — 4 bytes at offset 64**

```asm
401224:  xor $0x4b1d3f29, %eax
401229:  sub $0x6e58d392, %eax
; result must == 0x5a7eab95
```

The 4-byte integer at `buf[64]` is XOR-ed with `0x4b1d3f29`, then `0x6e58d392` is subtracted. The result must equal `0x5a7eab95`. Reversing this:

```
x ^ 0x4b1d3f29 = 0x5a7eab95 + 0x6e58d392 = 0xc8d77f27
x = 0xc8d77f27 ^ 0x4b1d3f29 = 0x83ca400e
```

**Check 2 — 4 bytes at offset 68**

```asm
40123e:  sub $0x6bdad9ef, %eax
401243:  xor $0x6f6f6f6f, %eax
; result must == 0x6fa08e7e
```

The 4-byte integer at `buf[68]` has `0x6bdad9ef` subtracted, then is XOR-ed with `0x6f6f6f6f`. The result must equal `0x6fa08e7e`. Reversing this:

```
(x - 0x6bdad9ef) ^ 0x6f6f6f6f = 0x6fa08e7e
x - 0x6bdad9ef = 0x6fa08e7e ^ 0x6f6f6f6f = 0x00cfe111
x = 0x00cfe111 + 0x6bdad9ef = 0x6caabb00
```

**Check 3 — 6 bytes at offset 72**

```asm
40127c:  xor $0x5c, %eax
; each byte XOR 0x5c must match stored pattern [3d, 2e, 3f, 3d, 38, 39]
```

Each byte at `buf[72..77]` is XOR-ed with `0x5c` and compared to the pattern `[0x3d, 0x2e, 0x3f, 0x3d, 0x38, 0x39]`. Reversing each byte:

```
0x3d ^ 0x5c = 0x61 = 'a'
0x2e ^ 0x5c = 0x72 = 'r'  (wait, actually...)
```

Working it out: `[0x3d ^ 0x5c, 0x2e ^ 0x5c, 0x3f ^ 0x5c, 0x3d ^ 0x5c, 0x38 ^ 0x5c, 0x39 ^ 0x5c]` = `arcade`

All three checks pass → the binary executes `cat flag.txt`.

### Step 5 — Verify with Python

```python
# Check 1
val1 = 0x83ca400e
assert ((val1 ^ 0x4b1d3f29) - 0x6e58d392) & 0xFFFFFFFF == 0x5a7eab95

# Check 2
val2 = 0x6caabb00
assert (((val2 - 0x6bdad9ef) & 0xFFFFFFFF) ^ 0x6f6f6f6f) == 0x6fa08e7e

# Check 3
pattern = bytes([0x3d, 0x2e, 0x3f, 0x3d, 0x38, 0x39])
suffix  = bytes([b ^ 0x5c for b in pattern])
assert suffix == b'arcade'

print("All checks verified!")
```

```
All checks verified!
```

---

## Solving the Challenge

### Step 6 — Build the payload

The correct 80-byte packet is:

| Bytes | Content |
|-------|---------|
| 0–63  | Zero padding |
| 64–67 | `\x0e\x40\xca\x83` (Check 1 value, little-endian) |
| 68–71 | `\x00\xbb\xaa\x6c` (Check 2 value, little-endian) |
| 72–77 | `arcade` (Check 3 suffix) |
| 78–79 | Zero padding |

```python
import sys, struct

buf = bytearray(80)
buf[64:68] = struct.pack('<I', 0x83ca400e)  # check 1
buf[68:72] = struct.pack('<I', 0x6caabb00)  # check 2
buf[72:78] = b'arcade'                      # check 3
sys.stdout.buffer.write(buf)
```

### Step 7 — Save the payload to a file

Piping Python directly into `nc` can introduce startup delay, allowing the watchdog timer to fire before all bytes arrive. Saving to a file first and redirecting it ensures the bytes are sent **instantly**:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
import sys, struct
buf = bytearray(80)
buf[64:68] = bytes([0x0e, 0x40, 0xca, 0x83])
buf[68:72] = bytes([0x00, 0xbb, 0xaa, 0x6c])
buf[72:78] = b'arcade'
sys.stdout.buffer.write(buf)
" > payload.bin
```

### Step 8 — Send the payload and capture the flag

```
┌──(zham㉿kali)-[~]
└─$ nc abort.aws.jerseyctf.com 1337 < payload.bin
jctf{$UccES5Fully_abOrt3D!_cOnGRATUl@t!0ns}
  NEON ARCADE CONTROL NODE v1.3.3
BOOT MESSAGE: Repair the cabinet link.
STATUS: Signal drift detected.
Insert maintenance packet:
[ SYSTEM OVERRIDE ACCEPTED ]
[ ACCESS LEVEL: ARCADE OPS ]
[ DUMPING VAULT CONTENTS ]
```

Flag obtained: **`jctf{$UccES5Fully_abOrt3D!_cOnGRATUl@t!0ns}`**

---

## One-liner

```bash
python3 -c "
import sys, struct
buf = bytearray(80)
buf[64:68] = bytes([0x0e,0x40,0xca,0x83])
buf[68:72] = bytes([0x00,0xbb,0xaa,0x6c])
buf[72:78] = b'arcade'
sys.stdout.buffer.write(buf)
" | nc abort.aws.jerseyctf.com 1337
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Identify binary type (ELF 64-bit, stripped) | ⭐ Easy |
| `strings` | Extract readable strings and spot `cat flag.txt` | ⭐ Easy |
| `objdump -d` | Disassemble the binary to read the check logic | ⭐⭐ Medium |
| Python | Reverse the arithmetic checks and build the payload | ⭐⭐ Medium |
| `nc` | Connect to the server and deliver the payload | ⭐ Easy |

---

## Key Takeaways

- **`strings` is your first tool** — before opening a disassembler, always run `strings` on an unknown binary. It revealed `cat flag.txt`, the timeout messages, and the success string, giving a clear picture of what the program does before reading a single line of assembly.
- **Reversing arithmetic is mechanical** — XOR and subtraction are completely invertible. Work backwards from the expected output: swap subtraction to addition, apply XOR again with the same key.
- **Piping vs. file redirection matters** — typing bytes interactively or adding Python startup time can lose the race against a watchdog timer. Always save your payload to a file and redirect it: `nc host port < payload.bin`.
- **Stripped binaries are still readable** — the absence of symbol names does not stop reverse engineering. The logic is still visible in the disassembly; you just need to name the functions yourself.
- **Little-endian byte order** — x86 stores multi-byte integers with the least significant byte first. When packing your integer values into the payload, always use `struct.pack('<I', value)` or `bytes([low, ..., high])`.
