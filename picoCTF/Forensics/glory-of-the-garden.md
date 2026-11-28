# Glory of the Garden — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Glory of the Garden | **Category** | Forensics |
| **Event** | picoCTF 2019 | **Difficulty** | Easy |
| **Author** | jedavis / Danny | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **File** | `garden.jpg` |

> **Flag:** `academy{more_than_m33ts_the_3y36c4fc727}`
> *"more than meets the eye"*. The flag was plain text hidden in the raw bytes of the image.

## 1. TL;DR
`garden.jpg` is a normal photo with the flag **appended as plain text** inside the file (after the image data). It isn't in the metadata, so `exiftool` finds nothing, but `strings` reads every printable sequence in the file and shows it directly.

## 2. The challenge
> *"This file contains more than it seems. Get the flag from garden.jpg"*

"Contains more than it seems" = extra data riding along inside the file, beyond the picture itself.

## 3. Analysis

### 3.1 Metadata check (came up empty)
```bash
exiftool garden.jpg      # ordinary JPEG fields, no flag
```
This was the right first instinct (it worked for information, CanYouSee, RED), but here the flag isn't in a metadata field. **Pivot needed.**

### 3.2 Read the raw strings
```bash
strings garden.jpg | grep -i academy
# academy{more_than_m33ts_the_3y36c4fc727}
```
`strings` prints every run of printable characters in the file, including text that isn't part of the image structure. The flag was appended as plain text, often after the JPEG's `FF D9` end-of-image marker.

### 3.3 Why it's there and not visible
A JPEG viewer stops reading at the `FF D9` end marker, so anything added after it is ignored by the viewer but still sits in the file. `strings` doesn't care about file structure, it reads the whole thing.

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Open the image | A normal garden photo, nothing visible |
| `exiftool` | No flag in metadata here (unlike information / CanYouSee) |
| LSB / stego tools | Over-thinking it. The data is plain text, not hidden in pixels |
| **`strings` + `grep` (chosen)** | Finds appended plaintext instantly |

> **The core insight:** when `exiftool` is empty but the file "contains more", the data is probably appended to the raw bytes. `strings` (and `binwalk` for whole files) is the next move.

## 5. The solver
**Command line (what we used):**
```bash
exiftool garden.jpg                       # nothing -> pivot
strings garden.jpg | grep -i academy      # flag
```
**If `grep academy` finds nothing, widen the net:**
```bash
strings garden.jpg | grep -iE "academy|flag|\{.*\}"   # also try encoded prefixes: YWNhZGVteX
binwalk garden.jpg                                     # look for appended/embedded files
binwalk -e garden.jpg                                  # extract them
```
> **Gotcha:** `strings` default minimum length is 4, which buries the flag in noise. Pipe to `grep` for the flag format, or sort by length. If the flag were **encoded** rather than plaintext, `grep academy` would miss it, so also grep for `YWNhZGVteX` (Base64) and `6163616465` (hex).

## 6. The run
```
$ strings garden.jpg | grep -i academy
academy{more_than_m33ts_the_3y36c4fc727}
```

## 7. Takeaways
**Solving:**
- **`strings | grep` is the cheapest first check on any file.** Plaintext flags appended to images/binaries are common.
- **`exiftool` empty ≠ dead end.** Pivot to `strings`, then `binwalk` for appended or embedded files.
- **Know the escalation order** for a "there's more here" image: look at it → `exiftool` → `strings` → `binwalk` → LSB/stego.
- **Carried forward:**
  - *DISKO 1 / Verify*: cheapest check first, `strings | grep`.
  - *information / CanYouSee*: those hid in metadata, this one didn't. Same instinct, different hiding spot, so the pivot mattered.

**Security:**
- Appending data after a file's end marker is a classic way to **smuggle payloads** inside innocent-looking images. Viewers and thumbnail generators ignore it, but the bytes are delivered.
- Detection: `binwalk` and trailing-data checks catch content after the expected end-of-file marker. Don't trust that a file is "just an image" because it opens as one.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Check **metadata** first | Habit from earlier metadata challenges |
| 2 | **Pivot to `strings`** when exiftool was empty | "Contains more than it seems" + no metadata flag |
| 3 | `grep academy` the output | Flag was appended as plain text |

**References:** picoCTF 2019: Glory of the Garden · Tools: `exiftool`, `strings`, `grep`, `binwalk`
