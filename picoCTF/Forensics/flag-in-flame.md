# Flag in Flame — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Flag in Flame | **Category** | Forensics |
| **Event** | picoMini by CMU-Africa | **Difficulty** | Easy |
| **Author** | Prince Niyonshuti N. | **Platform** | CyLab Academy |
| **Solved** | 2026-11-27 | **File** | `logs.txt` (1.59 MB, one block of text) |

> **Flag:** `academy{forensics_analysis_is_amazing_8dbd3133}`
> *"forensics analysis is amazing"*. It took three layers: Base64, then an image, then hex.

## 1. TL;DR
The "log file" isn't a log. It's one huge **Base64** string. Decoding it gives a **PNG image** (an illustrated hacker at a laptop). Printed along the bottom of the image is a long string of **hex**. Decoding that hex gives the flag. The flag is hidden three layers deep: **Base64 → PNG → hex → ASCII**.

## 2. The challenge
> *"The SOC team discovered a suspiciously large log file after a recent breach... they found an enormous block of encoded text instead of typical logs... reveal the real purpose of it."*

Three hints: **"large"**, **"encoded text"**, and **"real purpose"**. The file is something else disguised as a log, and it's big because it holds a whole file.

## 3. Analysis

### 3.1 Is it a log at all?
```bash
wc -c logs.txt          # 1592148  -> ~1.5 MB, no line structure
head -c 80 logs.txt
# iVBORw0KGgoAAAANSUhEUgAAA4AAAASACAIAAAAh8bSOAAEAAElEQVR4nOz919MsyZUniP3OcY+IFJ+6
tail -c 20 logs.txt
# 4QAAAABJRU5ErkJggg==
```
There are no timestamps, no log levels and no line breaks. Every character is in the Base64 alphabet and the file ends with `==` padding. **It's Base64.**

### 3.2 Recognise the file type before decoding
| Base64 prefix / suffix | Decoded meaning |
|---|---|
| `iVBORw0KGgo` (start) | `89 50 4E 47 0D 0A 1A 0A`, the **PNG** file signature |
| `SUhEUg` | `IHDR`, PNG's first chunk |
| `…JRU5ErkJggg==` (end) | `IEND` + CRC, the end of the PNG. Mid-stream patterns change with byte alignment, but the file always starts at offset 0, so the prefix is reliable |

You can tell it's a PNG **while it's still encoded**, the same idea as the `YWNhZGVteX` = `academy{` trick from Riddle Registry.

### 3.3 Decode
```bash
base64 -d logs.txt > decoded.png
file decoded.png
# PNG image data, 896 x 1152, 8-bit/color RGB, non-interlaced
```
1,592,148 Base64 characters decode to 1,194,109 bytes (Base64 turns every 3 bytes into 4 characters). The image is complete, starting with the `PNG` signature and ending with `IEND`.

### 3.4 Look at the image
The PNG is an illustration of a hooded hacker at a laptop. Along the bottom edge is a line of text that doesn't belong to the artwork:
```
61636164656D797B666F72656E736963735F616E616C797369735F69735F616D617A696E675F38646264333133337D
```

### 3.5 Recognise the encoding, again
| Clue | What it tells you |
|---|---|
| Only `0–9` and `A–F` | Hexadecimal |
| 94 characters (even length) | Whole bytes, 2 hex characters each |
| Starts with `61 63 61 64 65 6D 79 7B` | ASCII `a c a d e m y {` |
| Ends with `7D` | ASCII `}` |

```bash
echo '6163...337D' | xxd -r -p
# academy{forensics_analysis_is_amazing_8dbd3133}
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Read the "log" in a text editor | It's 1.5 MB on one line with nothing readable in it |
| `grep academy logs.txt` / `grep YWNhZGVteX logs.txt` | The flag isn't even in the Base64. It's a picture *inside* the decoded file |
| `strings` / `exiftool` on the decoded PNG | The flag is drawn in the pixels, not stored as text or metadata |
| Hunt for LSB stego / appended data in the PNG | Unnecessary. The data ends cleanly at `IEND`, and the clue is visible on screen |
| **Base64 → recognise PNG → view it → hex-decode the visible string (chosen)** | CyberChef **From Base64 → Render Image**, then **From Hex** |

> **The core insight:** each layer tells you what the next layer is. The `iVBORw0KGgo` prefix means PNG, so open it as an image. A string of only `0–9A–F` in the image means hex, so decode it to ASCII. Keep identifying and decoding until you reach readable text.

## 5. The solver

**CyberChef recipes (what we used):**
```
Step 1: From Base64  →  Render Image          (shows the PNG and the hex line)
Step 2: From Hex (Delimiter: Auto)            (paste the hex line, get the flag)
```

**Command-line equivalent:**
```bash
base64 -d logs.txt > decoded.png && xdg-open decoded.png     # read the hex line at the bottom
echo '<hex line from image>' | xxd -r -p                     # -> flag
```
> **Gotcha:** the hex string is in the image as **pixels**, so you can't copy it. You have to type it out by hand (or OCR it, e.g. `tesseract decoded.png - | grep -oE '[0-9A-F]{40,}'`). One mistyped character breaks the decode. Check that the length is even and the result is printable ASCII. A typo usually shows up as a garbled character in the middle of the flag.

## 6. The run
```
From Base64 → Render Image   : 896×1152 PNG, hex line along the bottom
From Hex                     : academy{forensics_analysis_is_amazing_8dbd3133}
```

## 7. Takeaways
**Solving:**
- **Layered encodings are normal. Keep peeling.** Identify, decode, identify the result, repeat until it's readable.
- **Learn file signatures in Base64 as well as hex:** `iVBORw0KGgo` = PNG, `/9j/` = JPEG, `UEsDB` = ZIP, `JVBERi0` = PDF, `R0lGOD` = GIF.
- **Know hex on sight:** `0–9A–F` only, even length. `61 63 61 64 65 6D 79 7B` is `academy{`.
- **Carried forward:**
  - *Binary Digits*: if an image has no text or metadata to find, **look at it**. The clue was visual again.
  - *Riddle Registry*: recognise the encoding from its character set and padding, and spot the known prefix while it's still encoded.

**Security (SOC angle):**
- A "log" with no log structure is a red flag. Attackers hide payloads, exfiltrated files and tooling in text files with harmless-sounding names to get past filters that only check extensions.
- Detection idea: flag log files with **no newlines**, a **pure Base64 character set**, or **high entropy**, and auto-decode the first bytes to see what file type is inside.
- Base64 and hex are encodings, not encryption. They hide nothing from anyone who recognises them.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Treat the "log" as **Base64**, not a log | No line structure, pure Base64 characters, `==` padding |
| 2 | Decode it as a **PNG image** | Prefix `iVBORw0KGgo` = PNG signature |
| 3 | **View** the image instead of searching its bytes | Lesson from Binary Digits: flags can be visual |
| 4 | **Hex-decode** the line of text in the image | Only `0–9A–F`, even length, starts with `6163616465…` |

**References:** picoMini by CMU-Africa: Flag in Flame · Tools: CyberChef (From Base64, Render Image, From Hex), `base64`, `file`, `xxd`
