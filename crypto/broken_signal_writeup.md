# Broken Signal — JerseyCTF Writeup

**Challenge:** Broken Signal  
**Category:** crypto  
**Difficulty:** Medium  
**Points:** 721  
**Solves:** 43  
**Challenge Author:** LoadinConfusion  
**Writeup by:** zham  
**Flag:** `JCTF{LOST_IN_COSMIC_STATIC}`

---

## Description

> Many years ago, a deep-space probe abruptly stopped responding to ground control. Its final transmissions were believed lost. Decades later, a fragment of the signal has been recovered from a degraded archival tape. A damaged reference sheet was recovered alongside the recording. Whatever the probe was transmitting...it might still be possible to recover it.

**Files provided:**
- `Unidentified.wav` — 58-second mono audio recording
- `Reference_Sheet.pdf` — damaged character frequency lookup table

---

## Background Knowledge (Read This First!)

### What is exiftool?

**exiftool** is a command-line tool that reads hidden metadata inside files — things like the file title, creation date, comments, and software used — stuff you can't see just by opening the file normally.

I ran it on both files first because metadata often leaks useful hints before you even touch the actual content.

### What is FFT (Fast Fourier Transform)?

**FFT** converts an audio signal from the time domain (a wave over time) into the frequency domain (which frequencies are present and how loud they are).

A simple way to think about it: if you press two piano keys at the same time, your ear hears a blended sound. FFT "listens" to that blended sound and tells you exactly which two notes are playing and how loud each one is.

This challenge encodes each character using **two frequencies played at the same time**, so FFT is exactly what I needed to figure out which character is being transmitted at each moment.

### What is Dual-Tone Encoding?

**Dual-tone encoding** means each character is sent by playing **one low frequency + one high frequency simultaneously**. This is the same concept used in old telephone keypads (DTMF tones).

The Reference Sheet gave me the lookup table:

```
         Col1     Col2     Col3     Col4     Col5
        1240Hz   2985Hz   3140Hz   5620Hz   6000Hz

252Hz     A        B        C        D        E
315Hz     F        G        H        I        J
515Hz     K        L        M        N        O
755Hz     P        Q        R        S        T
990Hz     U        V        W        X        Y
1530Hz    Z        {        }        .        _
```

**Example:** To transmit `J` → play **315 Hz + 6000 Hz** at the same time.
**Example:** To transmit `{` → play **1530 Hz + 2985 Hz** at the same time.

### What is scipy and numpy?

**numpy** is a Python library for fast math on large arrays — useful because audio files are just thousands of numbers.

**scipy** is built on top of numpy and gives me:
- `wavfile.read` — to load the `.wav` file into Python
- `fft` / `fftfreq` — to run FFT and get the frequencies out

Both are already on Kali so I didn't need to install anything.

---

## Solution — Step by Step

### Step 1 — Navigate to the downloads folder

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

---

### Step 2 — Check the metadata with exiftool

I ran exiftool on both files to see if there was anything hidden.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool Unidentified.wav
```

```
Title                           : spookysaturn
Duration                        : 0:00:58
Sample Rate                     : 44100
Num Channels                    : 1
Bits Per Sample                 : 16
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool Reference_Sheet.pdf
```

```
Title                           : Reference Sheet
Producer                        : Skia/PDF m148 Google Docs Renderer
Page Count                      : 1
```

The title `spookysaturn` stuck out to me — it fits the space/cosmic theme hinted at in the challenge description. Nothing else suspicious in the metadata, so I moved on to the audio itself.

---

### Step 3 — Read the Reference Sheet

Opening `Reference_Sheet.pdf` showed the character grid above. I noticed the last row at **1530 Hz** contains the special characters `Z { } . _` — including the flag brackets I'd need to find.

This confirmed the audio uses dual-tone encoding, and I had everything I needed to write a decoder.

---

### Step 4 — Write the decode script

I created the script using nano:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano decode.py
```

I pasted this script inside (**Ctrl+Shift+V** to paste in nano):

```python
# ── IMPORTS ──────────────────────────────────────────────────────────────────

import numpy as np
# numpy: I use this for fast math on the audio sample arrays

from scipy.io import wavfile
# wavfile: lets me load the .wav file into Python as a number array

from scipy.fft import fft, fftfreq
# fft     : runs Fast Fourier Transform — converts audio to frequency data
# fftfreq : gives me the actual Hz values matching each FFT output position

import warnings
warnings.filterwarnings('ignore')
# Hides non-critical warnings so the output stays clean


# ── LOAD THE AUDIO ───────────────────────────────────────────────────────────

sr, data = wavfile.read('Unidentified.wav')
# sr   = sample rate (44100 samples per second)
# data = the audio as a long array of numbers

data = data.astype(float)
# Convert to float for more accurate FFT math


# ── DEFINE THE FREQUENCY GRID ────────────────────────────────────────────────

row_freqs = [252, 315, 515, 755, 990, 1530]
# The 6 LOW frequencies — each one corresponds to a ROW in the grid
# 252Hz = row 1 (A-E), 1530Hz = row 6 (Z { } . _)

col_freqs = [1240, 2985, 3140, 5620, 6000]
# The 5 HIGH frequencies — each one corresponds to a COLUMN in the grid

grid = [
    list('ABCDE'),   # row 1 — 252Hz
    list('FGHIJ'),   # row 2 — 315Hz
    list('KLMNO'),   # row 3 — 515Hz
    list('PQRST'),   # row 4 — 755Hz
    list('UVWXY'),   # row 5 — 990Hz
    list('Z{}._'),   # row 6 — 1530Hz  ← { } _ are here
]
# grid[row][col] gives me the character at that position


# ── SLIDING WINDOW SETUP ─────────────────────────────────────────────────────

window_size = int(sr * 0.05)
# I analyze the audio 50ms at a time
# 44100 samples/sec × 0.05 sec = 2205 samples per window

n_windows = len(data) // window_size
# Total number of 50ms windows in the whole audio file


# ── STATE TRACKING ───────────────────────────────────────────────────────────

prev_ch = None
# Tracks what character I detected in the previous window
# Lets me group repeated windows into one symbol

char_start = 0
# Remembers which window the current character started at
# I use this to calculate the timestamp in seconds

char_dur = 0
# Counts how many windows in a row detected the same character
# More windows = longer duration = more likely a real symbol, not noise

results = []
# My final list of detected symbols: [(character, start_time, duration), ...]


# ── MAIN DECODING LOOP ───────────────────────────────────────────────────────

for i in range(n_windows):
    # Go through every 50ms window in the audio

    seg = data[i*window_size:(i+1)*window_size]
    # Grab this window's audio samples

    if np.max(np.abs(seg)) < 500:
        # If the loudest sample is very quiet → silence, nothing transmitted here
        if prev_ch and char_dur >= 3:
            results.append((prev_ch, char_start*0.05, char_dur*0.05))
            # Save the character I was tracking before this silence
        prev_ch = None
        char_dur = 0
        # Reset — the previous symbol has ended
        continue

    # ── RUN FFT ──────────────────────────────────────────────────────────────

    f = fftfreq(len(seg), 1/sr)
    # The list of Hz values that map to each FFT output bin

    F = np.abs(fft(seg))
    # Run FFT on this window — F[i] = amplitude of frequency f[i]

    # ── SEPARATE LOW AND HIGH BANDS ──────────────────────────────────────────

    low_mask = (f > 100) & (f < 1600)
    # I only look at 100–1600Hz for the row frequency

    high_mask = (f > 1100) & (f < 8000)
    # I only look at 1100–8000Hz for the column frequency

    low_dom = f[low_mask][np.argmax(F[low_mask])]
    # The frequency with the highest amplitude in the low band → my row guess

    high_dom = f[high_mask][np.argmax(F[high_mask])]
    # Same for the high band → my column guess

    low_amp = np.max(F[low_mask])
    # How strong is the dominant low frequency

    high_amp = np.max(F[high_mask])
    # How strong is the dominant high frequency

    total = np.max(F[f > 0])
    # Strongest frequency anywhere in the spectrum — used as a reference

    # ── MATCH TO GRID ────────────────────────────────────────────────────────

    row = min(range(6), key=lambda j: abs(row_freqs[j]-low_dom))
    # Find which row frequency is closest to what I detected

    col = min(range(5), key=lambda j: abs(col_freqs[j]-high_dom))
    # Find which column frequency is closest to what I detected

    row_ok = abs(row_freqs[row]-low_dom) < 120
    # Only accept if the matched row is within 120Hz — further = noise

    col_ok = abs(col_freqs[col]-high_dom) < 500
    # Only accept if the matched column is within 500Hz

    strong = low_amp/total > 0.2 and high_amp/total > 0.15
    # Both tones must be strong enough relative to the full signal
    # I lowered this from 0.4/0.3 to catch the degraded 1530Hz signals

    # ── SAVE THE CHARACTER ───────────────────────────────────────────────────

    if row_ok and col_ok and strong:
        ch = grid[row][col]
        # Look up the character using matched row and column

        if ch == prev_ch:
            char_dur += 1
            # Same character — extend the duration counter

        else:
            if prev_ch and char_dur >= 3:
                results.append((prev_ch, char_start*0.05, char_dur*0.05))
            prev_ch = ch
            char_start = i
            char_dur = 1
            # New character — start tracking it

    else:
        if prev_ch and char_dur >= 3:
            results.append((prev_ch, char_start*0.05, char_dur*0.05))
        prev_ch = None
        char_dur = 0


# ── COLLAPSE DUPLICATES ───────────────────────────────────────────────────────

collapsed = []
for ch, t, dur in results:
    if dur >= 0.25:
        # Only keep symbols that lasted at least 0.25s — shorter ones are noise
        if not collapsed or collapsed[-1][0] != ch:
            collapsed.append((ch, t, dur))
            # Skip if it's the same as the character I just added


# ── PRINT RESULTS ────────────────────────────────────────────────────────────

print("=== COLLAPSED MESSAGE ===")
for ch, t, dur in collapsed:
    print(f"  t={t:.1f}s: '{ch}' ({dur:.2f}s)")

msg = ''.join(r[0] for r in collapsed)
print("\nMessage:", msg)

if '{' in msg and '}' in msg:
    start = msg.index('{')
    end = msg.index('}')
    print("FLAG: JCTF" + msg[start:end+1])
else:
    print("\nBrackets not found.")
    print("Raw message for manual reading:", msg)
```

After pasting:
- Press **Ctrl+O** → **Enter** to save
- Press **Ctrl+X** to exit

---

### Step 5 — Run the script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 decode.py
```

**Output I got:**

```
=== COLLAPSED MESSAGE ===
  t=6.0s:  'J' (0.90s)
  t=7.1s:  'C' (0.90s)
  t=8.2s:  'T' (0.80s)
  t=9.2s:  'F' (0.80s)
  t=10.2s: 'V' (0.45s)
  t=11.3s: 'L' (0.80s)
  t=12.4s: 'O' (0.40s)
  t=13.4s: 'S' (0.80s)
  t=14.5s: 'T' (0.35s)
  t=15.5s: 'Y' (0.50s)
  t=16.6s: 'I' (0.35s)
  t=17.8s: 'N' (0.50s)
  t=18.8s: 'Y' (0.65s)
  t=20.1s: 'C' (0.50s)
  t=20.8s: 'O' (0.80s)
  t=21.8s: 'S' (0.25s)
  t=23.1s: 'M' (0.45s)
  t=24.1s: 'I' (0.60s)
  t=25.2s: 'C' (0.30s)
  t=26.2s: 'Y' (0.30s)
  t=27.1s: 'S' (0.80s)
  t=28.1s: 'T' (0.80s)
  t=29.7s: 'A' (0.25s)
  t=30.2s: 'T' (0.85s)
  t=31.2s: 'I' (0.80s)
  t=32.3s: 'C' (0.80s)
  t=33.4s: 'W' (0.80s)
  t=41.1s: 'U' (0.40s)

Message: JCTFVLOSTYINYCOSMICYSTATICWU

Brackets not found.
Raw message for manual reading: JCTFVLOSTYINYCOSMICYSTATICWU
```

---

### Step 6 — Figure out the missing brackets

The flag brackets `{`, `}`, and `_` weren't showing up. I looked at the grid and noticed something — those three characters are all in **row 6 at 1530 Hz**. Because the signal was degraded, my script was reading that row as **row 5 at 990 Hz** instead — same column, just one row off.

| What I got | What it actually is | Both share |
|---|---|---|
| `V` (row 5, col 2) | `{` (row 6, col 2) | column 2 — 2985 Hz |
| `W` (row 5, col 3) | `}` (row 6, col 3) | column 3 — 3140 Hz |
| `Y` (row 5, col 5) | `_` (row 6, col 5) | column 5 — 6000 Hz |

Once I substituted those in, the message became clear:

```
Raw:  J C T F V  L O S T  Y  I N  Y  C O S M I C  Y  S T A T I C  W
Real: J C T F {  L O S T  _  I N  _  C O S M I C  _  S T A T I C  }
```

That reads: **JCTF{LOST_IN_COSMIC_STATIC}**

---

## Flag

```
JCTF{LOST_IN_COSMIC_STATIC}
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `exiftool` | Checked metadata of the WAV and PDF files | ⭐ Easy |
| `nano` | Wrote and edited the decode script | ⭐ Easy |
| `python3` | Ran the decode script | ⭐ Easy |
| `numpy` | Handled the audio sample array math | ⭐⭐ Medium |
| `scipy` | Loaded the WAV file and ran FFT analysis | ⭐⭐ Medium |

---

## Key Takeaways

- Always run `exiftool` first — metadata often hides useful context before you even touch the file content
- Dual-tone audio encoding works like a phone keypad — two simultaneous frequencies identify one character
- FFT lets you "see" which frequencies are playing at any moment in the audio
- When the signal is degraded, loosen your detection thresholds or the weak signals won't register
- If flag brackets are missing, check adjacent rows in the grid — the row frequency might be off by one due to signal degradation
- The challenge name "Broken Signal" was a direct hint that the frequencies would be shifted or degraded
