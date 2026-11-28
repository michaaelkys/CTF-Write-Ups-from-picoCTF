# CanYouSee — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | CanYouSee | **Category** | Forensics |
| **Event** | picoCTF 2024 | **Difficulty** | Easy |
| **Author** | Mubarak Mikail | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **File** | `unknown.zip` → `ukn_reality.jpg` (4308×2875, 2.3 MB) |

> **Flag:** `academy{ME74D47A_HIDD3N_aebe8c0f}`
> *"metadata hidden"*. You can't see it in the photo, but it's in the file's data.

## 1. TL;DR
The zip contains a normal-looking JPEG photo. Nothing is visible in the image. Its metadata has an **`Attribution URL`** field holding a **Base64** string instead of a URL, and decoding it gives the flag.

## 2. The challenge
> *"How about some hide and seek?"*

The title asks **"Can You See"**. The answer is no, not by looking at the image. The flag is somewhere the photo viewer doesn't show.

## 3. Analysis

### 3.1 Unpack and look
```bash
unzip unknown.zip            # inflating: ukn_reality.jpg
xdg-open ukn_reality.jpg     # an ordinary photo, no visible flag
```

### 3.2 Cheapest check: metadata
```bash
exiftool ukn_reality.jpg
```
```
File Type        : JPEG
XMP Toolkit      : Image::ExifTool 12.76
Attribution URL  : YWNhZGVteXtNRTc0RDQ3QV9ISUREM05fYWViZThjMGZ9Cg==
Image Size       : 4308x2875
```
| Clue | Meaning |
|---|---|
| **`Attribution URL`** isn't a URL | A field meant for a web link holds random-looking text |
| `XMP Toolkit : Image::ExifTool` | Someone **edited the metadata** with ExifTool, so it was added on purpose |
| Starts **`YWNhZGVteX`**, ends `==` | `academy{` in **Base64** (not hex, which only uses `0–9A–F`) |

### 3.3 Decode
```bash
echo 'YWNhZGVteXtNRTc0RDQ3QV9ISUREM05fYWViZThjMGZ9Cg==' | base64 -d
# academy{ME74D47A_HIDD3N_aebe8c0f}
```
(The decoded value ends with a newline `\n`, which is the `Cg==` at the end of the Base64.)

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Look at the photo | Nothing visible, which is the point of "Can You See" |
| LSB / stego tools (RED-style) | Unnecessary. There's no instruction pointing to pixel-level hiding |
| `strings \| grep academy` | The flag is Base64-encoded, so plain `academy` never appears |
| **`exiftool` → spot the odd field → Base64 decode (chosen)** | Two commands |

> **The core insight:** this is Riddle Registry again with a different file type. The metadata field that doesn't match its purpose (a "URL" that isn't a URL) is the one to look at.

## 5. The solver
**Command line (what we used):**
```bash
unzip unknown.zip
exiftool ukn_reality.jpg
echo 'YWNhZGVteXtNRTc0RDQ3QV9ISUREM05fYWViZThjMGZ9Cg==' | base64 -d
```
**One-liner:**
```bash
exiftool -s3 -AttributionURL ukn_reality.jpg | base64 -d
```
> **Gotcha:** this is the third time a Base64 string has been mistaken for hex (RED, Ph4nt0m, here). Quick test: **lower-case letters or `=` at the end → Base64. Only `0–9` and `A–F` → hex.** And `YWNhZGVteX` at the start is always `academy{`.

## 6. The run
```
$ exiftool -s3 -AttributionURL ukn_reality.jpg | base64 -d
academy{ME74D47A_HIDD3N_aebe8c0f}
```

## 7. Takeaways
**Solving:**
- **Run `exiftool` on every image and document first.** It's the cheapest check, and it has found the flag or the next clue in three challenges now.
- **Compare each field's value with its name:** a "URL" with no `http`, an "Author" that isn't a name, a "Poem" that doesn't belong in a PNG.
- **`XMP Toolkit: Image::ExifTool`** means the metadata was edited with ExifTool, which is a hint that something was added on purpose.
- **Carried forward:**
  - *Riddle Registry*: odd metadata field + Base64 (`YWNhZGVteX`). Same method on a JPEG.
  - *RED*: metadata again. That time it held instructions, this time the flag itself.
  - *DISKO 1*: cheapest check first, so no stego tools needed.

**Security (SOC angle):**
- Photos carry a lot of metadata: camera model, GPS coordinates, software, author. Leaked metadata has exposed people's locations and identities. Strip it before publishing (`exiftool -all= photo.jpg`).
- Metadata fields are also used to **smuggle data** or payloads past filters that only check image content. Analysts should check metadata for encoded or oversized values.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Skip visual inspection, check **metadata** | Photo looks normal, and the title asks "Can You See" |
| 2 | Focus on **`Attribution URL`** | The value isn't a URL. `XMP Toolkit: ExifTool` shows the metadata was edited |
| 3 | **Base64-decode** it (not hex) | `YWNhZGVteX` prefix, lower-case letters, `==` padding |

**References:** picoCTF 2024: CanYouSee · Tools: `unzip`, `exiftool`, `base64`
