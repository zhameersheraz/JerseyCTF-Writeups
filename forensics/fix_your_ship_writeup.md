# Fix Your Ship — JerseyCTF Writeup

**Challenge:** Fix Your Ship  
**Category:** forensics  
**Difficulty:** Easy  
**Points:** 446  
**Solves:** 53  
**Flag:** `jctf{starship_voyager_apollo}`

---

## Description

> Your ship has crashed landed and you need to fix the engine but the files to access the schematics have been corrupted and there hexadecimal bits have been changed, luckily you remember the first file is a png which should help open the ships schematics give clues on how to fix the other files.

**Attachments:** `file1`, `file2`, `file3`

---

## Background Knowledge (Read This First!)

### What are Magic Bytes (File Signatures)?

Every file format begins with a specific sequence of bytes called a **magic number** or **file signature**. These bytes identify the format to the OS and applications — not the file extension. When these bytes are corrupted, the file becomes unreadable even though the rest of the data is intact.

| Format | Magic Bytes (hex) | Notes |
|--------|-------------------|-------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG....` |
| JPEG | `FF D8 FF` + end `FF D9` | SOI and EOI markers |
| MP4 | `xx xx xx xx 66 74 79 70` | Size + `ftyp` box |

### How to Inspect Raw Bytes

Use `od` to view the hex bytes of a file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ od -A x -t x1z file1 | head -3
```

---

## Solution — Step by Step

### Step 1 — Inspect the Raw Headers

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ od -A x -t x1z file1 | head -3
000000 89 00 00 00 0d 0a 1a 0a 00 00 00 0d 49 48 44 52  >............IHDR<

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ od -A x -t x1z file2 | head -3
000000 00 00 00 e0 00 10 4a 46 49 46 00 01 01 01 00 64  >......JFIF.....d<

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ od -A x -t x1z file3 | head -3
000000 00 00 00 20 70 66 79 74 69 73 6f 6d 00 00 02 00  >... pfytisom....<
```

Comparing against correct magic bytes:

- **file1:** Bytes 1–3 are `00 00 00` — should be `50 4E 47` (`PNG`)
- **file2:** Bytes 0–2 are `00 00 00` — should be `FF D8 FF`
- **file3:** Bytes 4–7 read `70 66 79 74` (`pfyt`) — should be `66 74 79 70` (`ftyp`)

### Step 2 — Fix file1 (PNG) and Read the Clue

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fix.py
```

```python
# Fix file1: restore PNG magic bytes
with open('file1', 'rb') as f:
    data = bytearray(f.read())

data[1] = 0x50  # P
data[2] = 0x4E  # N
data[3] = 0x47  # G

with open('file1_fixed.png', 'wb') as f:
    f.write(data)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fix.py

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ display file1_fixed.png &
```

Opening `file1_fixed.png` reveals a ship schematic with two key pieces of information:
- **Top right corner:** `jctf{starship` — the first flag segment
- **Bottom caption:** *"second file's first hex digit is FF and the last digit is D9"*

This confirms file2 is a JPEG (`FF D8 FF` start, `FF D9` end marker).

### Step 3 — Fix file2 (JPEG)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fix2.py
```

```python
# Fix file2: restore JPEG header and end marker
with open('file2', 'rb') as f:
    data = bytearray(f.read())

# Fix start
data[0] = 0xFF
data[1] = 0xD8
data[2] = 0xFF

# Fix end marker (clue from file1)
data[-2] = 0xFF
data[-1] = 0xD9

with open('file2_fixed.jpg', 'wb') as f:
    f.write(data)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fix2.py

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ display file2_fixed.jpg &
```

Opening `file2_fixed.jpg` shows a spaceship engine room schematic. Top right text:

```
_voyager_
```

### Step 4 — Fix file3 (MP4)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fix3.py
```

```python
# Fix file3: restore 'ftyp' box (bytes were transposed)
with open('file3', 'rb') as f:
    data = bytearray(f.read())

data[4] = 0x66  # f
data[5] = 0x74  # t
data[6] = 0x79  # y
data[7] = 0x70  # p

with open('file3_fixed.mp4', 'wb') as f:
    f.write(data)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fix3.py
```

The fixed MP4 is a short video. Extracting the last frame:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano getframe.py
```

```python
import cv2
cap = cv2.VideoCapture('file3_fixed.mp4')
total = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
cap.set(cv2.CAP_PROP_POS_FRAMES, total - 1)
ret, frame = cap.read()
cv2.imwrite('frame_last.jpg', frame)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 getframe.py

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ display frame_last.jpg &
```

The last frame displays green text:

```
apollo}
```

### Step 5 — Assemble the Flag

| File | Type | Flag Segment |
|------|------|--------------|
| file1 | PNG | `jctf{starship` |
| file2 | JPEG | `_voyager_` |
| file3 | MP4 | `apollo}` |

---

## Flag

```
jctf{starship_voyager_apollo}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `od` | Inspect raw hex bytes of each file | ⭐ Easy |
| `python3` | Patch the corrupted magic bytes | ⭐ Easy |
| `display` (ImageMagick) | View the fixed PNG and JPEG files | ⭐ Easy |
| `opencv (cv2)` | Extract frames from the fixed MP4 | ⭐ Easy |

---

## Key Takeaways

- **File extensions lie; magic bytes don't.** Always inspect the raw hex header to determine the true file format.
- **The first fixed file gave clues for the others.** The PNG contained both a flag segment and explicit instructions for repairing file2 — always fully read recovered files before moving on.
- **MP4 corruption can be subtle.** The `ftyp` box identifier being byte-swapped (`pfyt` → `ftyp`) is easy to miss if you only check the first few bytes.
- **Video frames carry flags.** Always extract multiple frames — the flag might only appear at the end of the clip.
