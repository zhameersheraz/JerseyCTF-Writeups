# Cold Wake — JerseyCTF Writeup

**Challenge:** Cold Wake  
**Category:** misc  
**Difficulty:** Easy  
**Points:** 446  
**Solves:** 53  
**Challenge Author:** Trent  
**Writeup by:** zham  
**Flag:** `JCTF{47291-80536-19408}`

---

## Description

> Three cassette tapes were recovered from cold storage beside a dormant pod at the outer edge of the Solar System. Before the nearby relay network went dark, an unseen operator split a launch authorization across three recordings and hid each segment differently. Recover the full code and bring the abandoned wake system online.

**Flag format:** `JCTF{XXXXX-XXXXX-XXXXX}`

**Attachment:** `01-cold-wake.zip`

---

## Background Knowledge (Read This First!)

### What is steghide?

**steghide** is a steganography tool that hides data inside image and audio files. The hidden data is password-protected and completely invisible to the naked eye. To extract it you need the correct passphrase.

```bash
steghide extract -sf image.jpg -p "password"
```

### What is stegseek?

**stegseek** is a lightning-fast steghide cracker. It takes a wordlist (like `rockyou.txt`) and tries every password automatically until it finds the one that works.

```bash
stegseek image.jpg /usr/share/wordlists/rockyou.txt
```

### What is EXIF metadata?

Every digital image file can carry hidden metadata called **EXIF** — information like camera model, GPS location, date taken, artist name, and custom comments. The `exiftool` command reads all of this metadata.

```bash
exiftool image.jpg
```

### What is speech recognition?

Audio files can contain spoken words. Python's `SpeechRecognition` library can send audio to Google's API and transcribe the spoken content into text — useful when a flag segment is read aloud in a recording.

```bash
pip3 install SpeechRecognition --break-system-packages
```

---

## Files Inside the ZIP

```
Tape1.jpg   (123 KB)  — JPEG image
Tape2.jpg   (11 MB)   — JPEG image
Tape3.jpg   (355 KB)  — JPEG image
```

Each tape hides one 5-digit segment of the flag using a different technique.

---

## Solution — Step by Step

### Step 1 — Extract the ZIP

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip 01-cold-wake.zip
Archive:  01-cold-wake.zip

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls *.jpg
Tape1.jpg  Tape2.jpg  Tape3.jpg
```

---

### Step 2 — Tape 1: EXIF Metadata

The first tape hides its segment directly in the JPEG comment field. Run `exiftool` to read all metadata:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool -v Tape1.jpg
  ExifToolVersion = 13.50
  FileName = Tape1.jpg
  FileSize = 123558
  FileType = JPEG
  MIMEType = image/jpeg
JPEG APP1 (298 bytes):
  ExifByteOrder = MM
  + [IFD0 directory with 8 entries]
  | 3)  Software = GRAVITY-WEAPON OS v1.3
  | 4)  ModifyDate = 1982:06:12 09:14:00
  | 5)  Artist = Cold War Orbital Research Division
JPEG COM (60 bytes):
  Comment = ORBITAL LAB ARCHIVE :: SINGULARITY INIT SEGMENT :: SEQ=47291
```

The JPEG comment field reveals the first segment directly:

```
ORBITAL LAB ARCHIVE :: SINGULARITY INIT SEGMENT :: SEQ=47291
```

**Tape 1 segment → `47291`**

---

### Step 3 — Tape 2 & Tape 3: Crack Steghide Password with stegseek

Both Tape2 and Tape3 have data hidden with steghide. Use stegseek to crack the password automatically against `rockyou.txt`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ stegseek Tape2.jpg /usr/share/wordlists/rockyou.txt

StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "galaxy"

[i] Original filename: "Tape2Paper_small.jpg".
[i] Extracting to "Tape2.jpg.out".

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ stegseek Tape3.jpg /usr/share/wordlists/rockyou.txt

StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "galaxy"

[i] Original filename: "Tape3_small.mp3".
[i] Extracting to "Tape3.jpg.out".
```

Both files share the same passphrase: **`galaxy`**

The extracted files are:
- `Tape2.jpg.out` — a JPEG image (259×200 px, grayscale)
- `Tape3.jpg.out` — an MP3 audio file (4.2 seconds, 56 kbps mono)

---

### Step 4 — Tape 2: Read the Image

Copy the extracted image and open it with ImageMagick:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cp Tape2.jpg.out /tmp/tape2out.jpg

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ display /tmp/tape2out.jpg &
```

The image shows a distressed paper texture with a large number printed on it:

```
80536
```

**Tape 2 segment → `80536`**

---

### Step 5 — Tape 3: Transcribe the Audio

Copy the extracted MP3 and convert it to WAV for analysis:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cp Tape3.jpg.out /tmp/tape3out.mp3

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sox /tmp/tape3out.mp3 /tmp/tape3.wav

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sox /tmp/tape3out.mp3 -n stat 2>&1 | head -5
Samples read:             93294
Length (seconds):      4.231020
Maximum amplitude:     0.907176
Rough   frequency:         1474
```

Generate a spectrogram to confirm the audio contains spoken words — 5 distinct speech segments are visible:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sox /tmp/tape3out.mp3 -n spectrogram -o /tmp/spec.png

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ display /tmp/spec.png &
```

Install SpeechRecognition and write a transcription script:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ pip3 install SpeechRecognition --break-system-packages
Successfully installed SpeechRecognition-3.16.0

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano /tmp/recognize.py
```

```python
import speech_recognition as sr
r = sr.Recognizer()
with sr.AudioFile('/tmp/tape3.wav') as src:
    audio = r.record(src)
try:
    print('Google:', r.recognize_google(audio))
except Exception as e:
    print('Error:', e)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 /tmp/recognize.py
Google: 1 9 4 0
```

Run again with `show_all=True` to catch all recognition candidates:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano /tmp/recognize2.py
```

```python
import speech_recognition as sr
r = sr.Recognizer()
r.energy_threshold = 100
r.pause_threshold = 2.0
with sr.AudioFile('/tmp/tape3.wav') as src:
    r.adjust_for_ambient_noise(src)
    audio = r.record(src)
try:
    result = r.recognize_google(audio, show_all=True)
    print('All results:', result)
except Exception as e:
    print('Error:', e)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 /tmp/recognize2.py
All results: {'alternative': [
  {'transcript': '9 4 0', 'confidence': 0.92789221},
  {'transcript': '940'},
  {'transcript': '9 4 0 8'},
  {'transcript': '9408'}
], 'final': True}
```

The first run returned `1 9 4 0` and the alternatives show `9 4 0 8`. Combining both results gives the full 5-digit number: **`19408`**

**Tape 3 segment → `19408`**

---

### Step 6 — Assemble the Flag

| Tape | Method | Segment |
|------|--------|---------|
| Tape 1 | EXIF comment field (`exiftool`) | `47291` |
| Tape 2 | Steghide (password: `galaxy`) → image | `80536` |
| Tape 3 | Steghide (password: `galaxy`) → MP3 speech | `19408` |

```
JCTF{47291-80536-19408}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `exiftool` | Read hidden EXIF metadata from Tape1 | ⭐ Easy |
| `stegseek` | Crack steghide password via wordlist brute-force | ⭐ Easy |
| `steghide` | Extract hidden files from Tape2 and Tape3 | ⭐ Easy |
| `display` (ImageMagick) | Open extracted image to visually read Tape2 number | ⭐ Easy |
| `sox` | Convert MP3 to WAV and generate spectrogram | ⭐ Easy |
| `SpeechRecognition` (Python) | Transcribe spoken digits from Tape3 audio | ⭐⭐ Medium |

---

## Key Takeaways

- **Always check EXIF metadata first** — challenge creators often hide flags or clues in JPEG comment, artist, or software fields. `exiftool -v` is your best friend.
- **stegseek makes steghide trivial** — instead of guessing passwords manually, stegseek tears through `rockyou.txt` in seconds. Always try it on any JPEG in a steganography challenge.
- **The same password can protect multiple files** — both Tape2 and Tape3 used `galaxy`. Once you crack one, try it on all remaining files immediately.
- **Flags can be hidden in audio** — spoken numbers in an MP3 are a valid steganography technique. When you extract an audio file, always check its spectrogram and try speech-to-text transcription.
- **Use `show_all=True` for ambiguous audio** — Google's speech recognition returns multiple candidates. Combining results from two separate recognition passes gave us the missing digit that completed the segment.
