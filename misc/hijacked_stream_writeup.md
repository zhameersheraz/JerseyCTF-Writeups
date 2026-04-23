# Hijacked Stream — JerseyCTF Writeup

**Challenge:** Hijacked Stream  
**Category:** misc  
**Difficulty:** Easy  
**Points:** 481  
**Solves:** 32  
**Authors:** Frank Crawford & Shawn Murray  
**Writeup by:** zham  
**Flag:** `jctf{vinylthon}`

---

## Description

> A NASA stream was hijacked by a bunch of music nerds. To keep suspicion down, they moved it to a new address and left a note. Can you find it and do some recon for us?

**Attachment:** `note.txt`

---

## Background Knowledge (Read This First!)

### What is Base64?

**Base64** is an encoding scheme that converts binary or text data into a string of printable ASCII characters. It is commonly used to transmit data over systems that only support text. You can always recognize Base64 by its character set (A–Z, a–z, 0–9, `+`, `/`) and optional `=` padding at the end.

For example:
```
aHR0cHM6Ly93anRicmFkaW8uY29t
```
Decoded → `https://wjtbradio.com`

To decode Base64 in Linux:
```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "aHR0cHM6Ly93anRicmFkaW8uY29t" | base64 -d
https://wjtbradio.com
```

### What is HLS (HTTP Live Streaming)?

**HLS** is a streaming protocol developed by Apple. Instead of one big video file, HLS breaks a stream into small `.mp4` segments and serves them via a playlist file called `index.m3u8`. The playlist tells your video player where to find each segment in order.

Example playlist:
```
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-STREAM-INF:BANDWIDTH=3007953,...
video1_stream.m3u8
```

### What is mediamtx?

**mediamtx** (formerly rtsp-simple-server) is an open-source media server that supports RTSP, HLS, WebRTC, and other streaming protocols. It serves live video and audio streams over HTTPS. When you find a mediamtx server, you can discover active streams by checking paths like `/stream2/index.m3u8`.

### What is ffmpeg?

**ffmpeg** is a powerful command-line tool for processing video and audio files. In this challenge it is used to extract still image frames from a downloaded video segment. HLS segments need the **init file** combined with a segment file before ffmpeg can decode them properly.

### What is feh?

**feh** is a lightweight image viewer for Linux. It opens image files directly in a window on your desktop. In this challenge, after extracting frames from the video, `feh` is used to open the `.jpg` frame and **visually read the flag** displayed on screen.

---

## Solution — Step by Step

### Step 1 — Decode the note

The downloaded `note.txt` contains a single Base64-encoded string:

```
aHR0cHM6Ly93anRicmFkaW8uY29t
```

Decode it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "aHR0cHM6Ly93anRicmFkaW8uY29t" | base64 -d
https://wjtbradio.com
```

The note points to **WJTB Radio** — NJIT's official college radio station. The challenge description mentions "music nerds" hijacking a NASA stream, which fits perfectly.

### Step 2 — Recon the website

Fix PATH first, then begin reconnaissance:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ export PATH=/usr/bin:/bin:$PATH

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl https://wjtbradio.com/robots.txt
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -s https://wjtbradio.com/ | grep -i "jctf"

```

No flag found in robots.txt or page source. The site runs on **nginx/1.24.0** and is built with **Next.js**.

### Step 3 — Find the hidden stream URL in JavaScript

The Next.js site bundles all URLs into JavaScript chunk files that are publicly accessible. Extract all URLs from the layout chunk:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -s "https://wjtbradio.com/_next/static/chunks/app/(pages)/layout-c4c2b901f7a4d10b.js" | \
pipe> grep -oP 'https?://[^"'\'')\s]+'
https://wjtbradio.com:8888/stream2/index.m3u8
https://wjtbradio.com:8000/stream1.mp3
https://wjtbradio.com/strapi
https://wjtbradio.com/strapi/api/
```

Port **8888** hosts a second hidden stream: `stream2/index.m3u8`. This is the "hijacked NASA stream" that was moved to a new address.

### Step 4 — Access the HLS stream

Port 8888 requires **HTTPS** (plain HTTP returns an error). Use `-k` to bypass the self-signed certificate warning:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -sk https://wjtbradio.com:8888/stream2/index.m3u8
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-INDEPENDENT-SEGMENTS

#EXT-X-MEDIA:TYPE="AUDIO",GROUP-ID="audio",NAME="audio2",AUTOSELECT=YES,DEFAULT=YES,URI="audio2_stream.m3u8"

#EXT-X-STREAM-INF:BANDWIDTH=3096148,AVERAGE-BANDWIDTH=3005459,CODECS="avc1.640028,mp4a.40.2",RESOLUTION=1920x1080,FRAME-RATE=23.976,AUDIO="audio"
video1_stream.m3u8
```

The response header confirms this is a **mediamtx** server serving a live 1080p HLS video stream.

### Step 5 — Download a video segment and extract frames

HLS segments require the **init file** combined with a segment file for ffmpeg to decode them. Get the latest segment name from the playlist:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ SEG=$(curl -sk https://wjtbradio.com:8888/stream2/video1_stream.m3u8 | grep "seg[0-9]*\.mp4" | tail -1)

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "Latest segment: $SEG"
Latest segment: da96d376321f_video1_seg9856.mp4
```

Download the init file and latest segment, then combine them:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -sk "https://wjtbradio.com:8888/stream2/da96d376321f_video1_init.mp4" -o /tmp/init.mp4

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ curl -sk "https://wjtbradio.com:8888/stream2/$SEG" -o /tmp/seg.mp4

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat /tmp/init.mp4 /tmp/seg.mp4 > /tmp/combined.mp4
```

Extract 5 frames from the combined video:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ffmpeg -i /tmp/combined.mp4 -vframes 5 /tmp/frame%d.jpg -y 2>/dev/null

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la /tmp/frame*.jpg
-rw-rw-r-- 1 zham zham 69480 Apr 20 07:42 /tmp/frame1.jpg
-rw-rw-r-- 1 zham zham 76877 Apr 20 07:42 /tmp/frame2.jpg
-rw-rw-r-- 1 zham zham 87040 Apr 20 07:42 /tmp/frame3.jpg
-rw-rw-r-- 1 zham zham 80431 Apr 20 07:42 /tmp/frame4.jpg
-rw-rw-r-- 1 zham zham 70943 Apr 20 07:42 /tmp/frame5.jpg
```

Copy the frame to home directory:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cp /tmp/frame1.jpg ~/frame1.jpg
```

### Step 6 — View the frame and read the flag

Open the extracted frame with `feh`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ feh /home/zham/frame1.jpg
```

The image shows the live video stream playing on screen. The flag `jctf{vinylthon}` is **visually displayed as text on the video** — the "hijacked NASA stream" was a live broadcast with the flag shown directly on screen.

---

## One-liner Grab Script

Instead of running each command one by one, we can put everything into a single script using `nano`, save it, and run it all at once.

Create the script:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano grab_frame.sh
```

Paste the following content inside nano:

```bash
#!/bin/bash
export PATH=/usr/bin:/bin:$PATH
SEG=$(curl -sk https://wjtbradio.com:8888/stream2/video1_stream.m3u8 | grep "seg[0-9]*\.mp4" | tail -1)
echo "[*] Latest segment: $SEG"
curl -sk "https://wjtbradio.com:8888/stream2/da96d376321f_video1_init.mp4" -o /tmp/init.mp4
curl -sk "https://wjtbradio.com:8888/stream2/$SEG" -o /tmp/seg.mp4
cat /tmp/init.mp4 /tmp/seg.mp4 > /tmp/combined.mp4
ffmpeg -i /tmp/combined.mp4 -vframes 5 /tmp/frame%d.jpg -y 2>/dev/null
echo "[*] Frames saved:"
ls -la /tmp/frame*.jpg
cp /tmp/frame1.jpg ~/frame1.jpg
echo "[*] Opening image to view flag..."
feh ~/frame1.jpg
```

Save and exit nano:
- Press **Ctrl+O** to save
- Press **Enter** to confirm the filename
- Press **Ctrl+X** to exit

Make the script executable and run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x grab_frame.sh && ./grab_frame.sh
[*] Latest segment: da96d376321f_video1_seg9856.mp4
[*] Frames saved:
-rw-rw-r-- 1 zham zham 69480 Apr 20 07:42 /tmp/frame1.jpg
-rw-rw-r-- 1 zham zham 76877 Apr 20 07:42 /tmp/frame2.jpg
-rw-rw-r-- 1 zham zham 87040 Apr 20 07:42 /tmp/frame3.jpg
-rw-rw-r-- 1 zham zham 80431 Apr 20 07:42 /tmp/frame4.jpg
-rw-rw-r-- 1 zham zham 70943 Apr 20 07:42 /tmp/frame5.jpg
[*] Opening image to view flag...
```

`feh` opens the frame image and the flag `jctf{vinylthon}` is visible on screen.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `base64 -d` | Decode the Base64-encoded note.txt | ⭐ Easy |
| `curl` | Fetch web pages, JS files, and stream playlists | ⭐ Easy |
| `grep` | Search for URLs in JavaScript source files | ⭐ Easy |
| `ffmpeg` | Combine HLS init + segment and extract video frames | ⭐⭐ Medium |
| `feh` | Open the extracted frame image to visually read the flag | ⭐ Easy |
| `nano` | Create and edit the grab script | ⭐ Easy |

---

## Key Takeaways

- **Base64 is not encryption** — it is just encoding. Always try decoding any suspicious-looking strings in challenge files.
- **JavaScript source files are goldmines** — Next.js and other frameworks bundle all URLs and endpoints into publicly accessible JS chunks. Always check them during recon.
- **Flags can be hidden anywhere** — not just in file metadata or HTTP headers, but displayed visually on a live video stream.
- **HLS streams need init + segment** — you cannot decode an HLS `.mp4` segment alone. Always combine it with the init file first before running ffmpeg.
- **Sometimes the simplest tool wins** — after all the scripting, the flag was found simply by opening an image and looking at it with `feh`.
