# RED — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | RED | **Category** | Forensics |
| **Event** | picoCTF 2025 | **Difficulty** | Easy |
| **Author** | Shuailin Pan (LeConjuror) | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **File** | `red.png` (128×128, RGBA, 792 bytes) |

> **Flag:** `academy{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}`
> *"red is the ultimate cure for sadness"*. The poem told us where to look, and the pixels held the answer.

## 1. TL;DR
`red.png` is a plain red square with a non-standard **`Poem`** field in its metadata. The poem is an **acrostic**: the first letter of each line spells **`CHECK LSB`**. Extracting the **least significant bit** of each pixel's R, G, B and A values gives a **Base64** string, which decodes to the flag.

## 2. The challenge
> *"RED, RED, RED, RED"*

Just an image and a title repeated four times. "Four" lines up with the **four colour channels** in an RGBA image (Red, Green, Blue, Alpha).

## 3. Analysis

### 3.1 Cheapest check first: metadata
```bash
exiftool red.png
```
```
Image Size  : 128x128
Color Type  : RGB with Alpha
Poem        : Crimson heart, vibrant and bold,.Hearts flutter at your sight..Evenings glow softly red,.
              Cherries burst with sweet life..Kisses linger with your warmth..Love deep as merlot..
              Scarlet leaves falling softly,.Bold in every stroke.
```
PNG has no standard **`Poem`** field. Someone added it on purpose, the same giveaway as the odd field in Riddle Registry.

### 3.2 Read the poem's structure, not its words
Split the poem at each `.` and take the first letter of each line:

| # | Line | Letter |
|---|---|---|
| 1 | **C**rimson heart, vibrant and bold, | C |
| 2 | **H**earts flutter at your sight. | H |
| 3 | **E**venings glow softly red, | E |
| 4 | **C**herries burst with sweet life. | C |
| 5 | **K**isses linger with your warmth. | K |
| 6 | **L**ove deep as merlot. | L |
| 7 | **S**carlet leaves falling softly, | S |
| 8 | **B**old in every stroke. | B |

**`CHECK LSB`**: an instruction pointing to least-significant-bit steganography.

### 3.3 What LSB steganography is
Each pixel stores 4 values (R, G, B, A) from 0 to 255. Changing the **last bit** of a value (e.g. `254` → `255`) changes the colour by 1/255, which you can't see. Taking that last bit from every channel of every pixel, in order, rebuilds a hidden byte stream while the image looks unchanged.

### 3.4 Extract the bits
CyberChef **Extract LSB** with the colour pattern **R, G, B, A** (all four channels, the "RED ×4" hint), bit 0, gives:
```
YWNhZGVteXtyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==
```

### 3.5 Identify the encoding
| Clue | Meaning |
|---|---|
| Starts with **`YWNhZGVteX`** | `academy{` in Base64 (the prefix from Riddle Registry) |
| `A–Z a–z 0–9`, ends `==` | Base64 alphabet + padding |

It **looks** like random characters, but it isn't hex. Hex only uses `0–9A–F`. The prefix gives it away immediately.

```bash
echo 'YWNhZGVteXtyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==' | base64 -d
# academy{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Look at the image | A solid red square. Nothing visible, unlike Binary Digits / Flag in Flame |
| `strings red.png \| grep academy` | Flag is spread across pixel bits, and encoded, so it never appears as bytes |
| Take the poem's words literally (cipher, keyword) | The meaning is in the **line structure**, not the words |
| Extract LSB from the **R channel only** | Misses most of the data. All four channels carry bits |
| **Read the acrostic → Extract LSB (RGBA) → Base64 decode (chosen)** | Three steps, each one pointed to by the previous clue |

> **The core insight:** the metadata didn't hold the flag. It held the **instructions**. Once you know a clue is a signpost, follow it to the method it names (LSB) and the parameters it hints at (four channels).

## 5. The solver
**CyberChef recipe (what we used):**
```
Extract LSB (Colour pattern: R, G, B, A · Pixel order: Row · Bit: 0)  →  From Base64
```
**Command-line equivalent:**
```bash
zsteg red.png                    # tries common LSB layouts, look for the b1,rgba row
zsteg -E b1,rgba,lsb,xy red.png | head -c 64 | base64 -d
```
> **Gotcha:** the result of an LSB extraction is just bytes. Identify what they are before decoding. We first called the output "hex", but `YWNhZGVteX` (with lower-case letters and `==` padding) marks it as **Base64**. Picking the wrong decoder would give nonsense.

## 6. The run
```
Extract LSB (RGBA)  →  YWNhZGVteXtyM2RfMXNfdGgzX3VsdDFtNHQzX2N1cjNfZjByXzU0ZG4zNTVffQ==
From Base64         →  academy{r3d_1s_th3_ult1m4t3_cur3_f0r_54dn355_}
```

## 7. Takeaways
**Solving:**
- **A metadata field that shouldn't exist is a clue,** even when it isn't the flag itself.
- **Check for acrostics:** first letters of lines (and last letters, or first letters of words) often spell instructions.
- **LSB steganography is the default suspect** for a plain-looking PNG/BMP with an alpha channel and nothing else obvious. Try `zsteg` (PNG/BMP) or CyberChef Extract LSB, and try different channel orders.
- **Read the title as a parameter:** "RED ×4" pointed to the four RGBA channels.
- **Carried forward:**
  - *Riddle Registry*: the odd metadata field, and the `YWNhZGVteX` = `academy{` Base64 prefix.
  - *DISKO 1*: cheapest check first. `exiftool` took seconds and gave us the roadmap.

**Security:**
- LSB steganography can hide data in images that look completely normal. It's used for covert channels and data exfiltration, and normal content filters never notice the change.
- Re-encoding or resizing an image destroys LSB data. Some platforms do this on purpose as a defence (like how social media recompresses uploads).

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Focus on the **`Poem`** metadata field | PNG has no standard Poem field |
| 2 | Read it as an **acrostic** | Line structure (split on `.`), first letters spell `CHECK LSB` |
| 3 | **Extract LSB from all four channels** | Acrostic + "RED, RED, RED, RED" + RGBA colour type |
| 4 | Decode as **Base64**, not hex | Output starts `YWNhZGVteX` and ends `==` |

**References:** picoCTF 2025: RED · Tools: exiftool, CyberChef (Extract LSB, From Base64), zsteg
