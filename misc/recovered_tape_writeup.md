# Recovered Tape — JerseyCTF Writeup

**Challenge:** Recovered Tape  
**Category:** misc  
**Difficulty:** Easy  
**Points:** 369  
**Solves:** 27  
**Challenge Author:** Frank Crawford  
**Writeup by:** zham  
**Flag:** `jctf{buy_0ne_g3t_0ne_fr33}`

---

## Description

> A shopping mall in Kansas City finally retired their '80s PA system, originally designed to work alongside looped music tapes for sale announcements. They want to archive its contents but can't figure out the messages. A short clip was salvaged from one of them.

**Attachment:** `clip.wav`

---

## Background Knowledge (Read This First!)

### What is a Spectrogram?

A **spectrogram** is a visual representation of the frequencies present in an audio signal over time. The horizontal axis is time, the vertical axis is frequency, and the color/brightness indicates the intensity (loudness) of each frequency at each moment. Spectrograms are essential in audio steganography challenges because hidden data often shows up as unusual frequency patterns invisible to the naked ear.

To generate a spectrogram in Linux using `sox`:
```bash
sox clip.wav -n spectrogram -o spectrogram.png
```

### What are Stereo Audio Channels?

A stereo `.wav` file contains **two independent audio channels**: left and right. In a normal recording both channels carry music. In this challenge, the **right channel** carries a completely separate hidden signal — a technique used to smuggle data inside audio without disturbing the main audio on the left channel.

To split channels in Linux using `sox`:
```bash
sox clip.wav -c 1 right_only.wav remix 2   # extract right channel
sox clip.wav -c 1 left_only.wav remix 1    # extract left channel
```

### What is FSK (Frequency Shift Keying)?

**FSK** is a digital modulation scheme that encodes binary data by switching between two distinct audio tones:
- **Mark tone** → binary `1`
- **Space tone** → binary `0`

FSK was the foundation of early dial-up modems (Bell 202, Bell 103). In this challenge, the hidden signal uses exactly **two tones — 1200 Hz and 2400 Hz** — which is the frequency pair used by the **Bell 202 modem standard** at 300 baud.

### What is minimodem?

**minimodem** is a command-line tool that acts as a software modem. It can both transmit and receive FSK-encoded data from audio files, simulating old modem protocols. Crucially, you can specify the exact mark and space frequencies manually, which is essential when the encoding is non-standard or reversed.

Install it on Kali:
```bash
sudo apt install minimodem
```

Basic usage to decode:
```bash
minimodem --rx 300 --mark 1200 --space 2400 -f right_only.wav
```

---

## Solution — Step by Step

### Step 1 — Inspect the Audio File

Start by examining the basic properties of the WAV file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file clip.wav
clip.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, stereo 44100 Hz
```

Key facts:
- **Stereo** (two channels)
- **44100 Hz** sample rate
- **16-bit** PCM
- Duration: ~17.5 seconds

The file is stereo — that's immediately suspicious. Always investigate both channels separately in a challenge.

### Step 2 — Generate a Spectrogram

Use `sox` to generate a spectrogram of the full file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sox clip.wav -n spectrogram -o spectrogram.png -x 3000 -y 1000 -z 120 -w hann
```

Open the image:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ feh spectrogram.png
```

The spectrogram shows two channels side by side. The **top half (left channel)** looks like normal music — continuous noise across multiple frequencies. The **bottom half (right channel)** is dramatically different: it shows a large **silent gap from about 8 to 9.5 seconds**, with a bright horizontal stripe at a very specific frequency. This is the hidden signal.

### Step 3 — Isolate the Right Channel

Extract the right channel as a standalone mono WAV file using `sox`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sox clip.wav -c 1 right_only.wav remix 2
```

Generate a zoomed spectrogram of just the right channel around the interesting time window:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sox right_only.wav -n spectrogram -o right_spec.png -x 2000 -y 500 -z 120
```

The spectrogram now clearly shows:
- The music **cuts out** completely around 8.25 seconds
- Two distinct **horizontal bands** appear at **1200 Hz and 2400 Hz**
- The bands alternate back and forth — binary data is being transmitted by switching between the two tones
- The signal ends around 9.25 seconds, then the music resumes

This is **FSK — Frequency Shift Keying**.

### Step 4 — Identify the Signal Parameters

Run a quick frequency analysis on the active section of the right channel using Python and `scipy`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import wave, numpy as np
with wave.open('clip.wav') as w:
    rate = w.getframerate()
    data = np.frombuffer(w.readframes(w.getnframes()), dtype=np.int16)
    right = data[1::2].astype(float)
# FFT of the active section (8.25-9.25s)
seg = right[int(8.25*rate):int(9.25*rate)]
fft = np.abs(np.fft.rfft(seg))
freqs = np.fft.rfftfreq(len(seg), 1/rate)
# Find top peaks
top = np.argsort(fft)[-5:]
for i in sorted(top):
    print(f'{freqs[i]:.1f} Hz: amplitude {fft[i]:.0f}')
"
1200.0 Hz: amplitude 46240604
2400.0 Hz: amplitude 78284103
```

Confirmed: exactly **1200 Hz** and **2400 Hz** — the classic Bell 202 FSK modem frequency pair.

### Step 5 — Install minimodem

If not already installed:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install minimodem
[sudo] password for zham:
...
Setting up minimodem (1.3.1+dfsg-1.1)...
```

### Step 6 — Decode the FSK Signal (Standard Attempt)

Try the standard Bell 202 configuration first — mark=1200 Hz, space=2400 Hz, at 300 baud:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ minimodem --rx 300 --mark 1200 --space 2400 -f right_only.wav 2>&1 | strings | grep -v "^###"
%r&'
```

Garbage output. The data isn't decoding correctly with standard settings.

### Step 7 — Decode the FSK Signal (Reversed Mark/Space)

The mark and space frequencies are **swapped** from the standard Bell 202 convention. This is the intentional twist — the PA system encoded data with **2400 Hz as the mark (1)** and **1200 Hz as the space (0)**. Flip the arguments:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ minimodem --rx 300 --mark 2400 --space 1200 -f right_only.wav 2>&1 | strings | grep -v "^###"
jctf{buy_0ne_g3t_0ne_fr33}
```

The flag appears clearly.

---

## One-liner Grab Script

Instead of running each command one by one, paste everything into a single script using `nano`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano decode_tape.sh
```

Paste the following content inside nano:

```bash
#!/bin/bash
export PATH=/usr/bin:/bin:$PATH

echo "[*] Extracting right channel..."
sox clip.wav -c 1 /tmp/right_only.wav remix 2

echo "[*] Decoding FSK signal (reversed mark/space, 300 baud)..."
FLAG=$(minimodem --rx 300 --mark 2400 --space 1200 -f /tmp/right_only.wav 2>/dev/null | \
       grep -oP 'jctf\{[^}]+\}')

echo "[*] Flag: $FLAG"
```

Save and exit nano:
- Press **Ctrl+O** to save
- Press **Enter** to confirm the filename
- Press **Ctrl+X** to exit

Make the script executable and run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x decode_tape.sh && ./decode_tape.sh
[*] Extracting right channel...
[*] Decoding FSK signal (reversed mark/space, 300 baud)...
[*] Flag: jctf{buy_0ne_g3t_0ne_fr33}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Check audio format and channel count | ⭐ Easy |
| `sox` | Extract right channel and generate spectrograms | ⭐⭐ Medium |
| `feh` | View spectrograms visually to identify hidden signal | ⭐ Easy |
| `python3` + `numpy` | FFT analysis to confirm exact frequencies | ⭐⭐ Medium |
| `minimodem` | Decode the FSK modem signal from the audio | ⭐⭐ Medium |

---

## Key Takeaways

- **Always check both channels of a stereo audio file** — the left and right channels are independent, and one can carry a completely different hidden signal while sounding normal.
- **Spectrograms reveal what ears cannot** — the FSK signal was invisible when listening but immediately obvious as alternating horizontal bands at 1200 Hz and 2400 Hz in the spectrogram.
- **Standard tools have non-standard configurations** — `minimodem` decodes Bell 202 by default with mark=1200, space=2400. The challenge swapped these, producing garbage output until the arguments were flipped.
- **Context is a hint** — the '80s PA system theme directly suggests old modem/FSK technology. The 1200/2400 Hz frequency pair is the exact Bell 202 modem standard that was common in the early 1980s.
- **The flag itself is thematic** — `jctf{buy_0ne_g3t_0ne_fr33}` is a "Buy One Get One Free" sale announcement, exactly what a shopping mall PA system would broadcast. The hidden message was a sale ad all along.
