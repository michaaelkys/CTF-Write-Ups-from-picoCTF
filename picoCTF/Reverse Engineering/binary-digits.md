# Binary Digits — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Binary Digits | **Category** | Forensics |
| **Event** | picoCTF 2026 | **Difficulty** | Easy |
| **Author** | Yahaya Meddy | **Platform** | CyLab Academy |
| **Solved** | 2026-11-27 | **File** | `digits.bin` (71,728 chars of `0`/`1`) |

> **Flag:** `academy{h1dd3n_1n_th3_b1n4ry_5a7375c5}`
> *"hidden in the binary"* — the noise was a picture all along.

## 1. TL;DR
`digits.bin` is not a binary file — it's a **text file** of ASCII `0` and `1` characters. Grouping them 8 at a time and packing them back into real bytes produces a **JPEG image** (`FF D8 FF E0 … JFIF`). The flag isn't in the file's text or metadata; it's **drawn in the pixels**, so the final step is simply rendering the image.

## 2. The challenge
> *"This file doesn't look like much... just a bunch of 1s and 0s. But maybe it's not just random noise. Can you recover anything meaningful from this?"*

```
1111111111011000111111111110000000000000000100000100101001000110...
```
The extension says `.bin`, but the content is printable text. **The description's hint — "not random noise" — says there's structure to decode, not entropy to crack.**

## 3. Analysis

### 3.1 Is it really 8-bit data?
```bash
tr -cd '01' < digits.bin | wc -c      # 71728  ->  71728 / 8 = 8966, no leftover bits
```
A length that divides cleanly by 8 is the first confirmation that this is bytes written out as bits.

### 3.2 Read the first bytes by eye
```
11111111 11011000 11111111 11100000 ...
   FF       D8       FF       E0
```
`FF D8 FF E0` is the **JPEG/JFIF magic** — recognisable before writing any code.

### 3.3 Convert and identify
In CyberChef, run **From Binary** (Delimiter: None) and **Save** the output as `out.bin`, then:
```bash
file out.bin
# JPEG image data, JFIF standard 1.01, ... baseline, precision 8, 800x500, components 3
```
Header: `ff d8 ff e0 00 10 4a 46 49 46` (`JFIF`) · Trailer: `ff d9` at the very last bytes → **a clean, complete JPEG with nothing appended**.

### 3.4 Where is the flag?
| Check | Result |
|---|---|
| `strings -n 6 out.bin` | Only JPEG quantization/Huffman table noise — no flag |
| `exiftool out.bin` | Plain JFIF metadata — no comment, no flag |
| Data after `FF D9` (EOI) | None — EOI is at offset 8964 of 8966 |
| **Render the image** | **Flag drawn in red text on a white 800×500 canvas** |

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Treat `digits.bin` as a binary (`file`, `binwalk` on the original) | It reports ASCII text — the real file is one decode layer down |
| `strings` / grep for `academy{` on the decoded file | Flag is pixels, not bytes — no text match exists |
| Hunt for stego (appended data, EXIF comments, LSB) | Nothing appended after `FF D9`, metadata clean — over-thinking an Easy challenge |
| **Decode bits → bytes, identify by magic, render (chosen)** | CyberChef **From Binary → Render Image** shows the flag in one step |

> **The core insight:** the encoding layer (`0`/`1` text) hides the *file type*. Once the bytes are restored, the magic `FF D8 FF E0` tells you exactly what to do next: open it as an image.

## 5. The solver

**CyberChef recipe (what we used):**
```
From Binary (Delimiter: None, Byte length: 8)  →  Render Image (Input format: Raw)
```

> **Gotcha:** the input must contain only `0`/`1`. A stray newline or space shifts every following byte by one bit and turns the whole file into garbage. CyberChef's From Binary ignores whitespace, but a hand-rolled converter may not.

## 6. The run
CyberChef output: 8,966 bytes, starting `FF D8 FF E0`. Rendered image (800×500, white background, red text):
```
academy{h1dd3n_1n_th3_b1n4ry_5a7375c5}
```

## 7. Takeaways
**Solving:**
- **The extension lies; the content doesn't.** A `.bin` full of printable `0`/`1` is an *encoding*, not a binary. Peel the layer first.
- **Check byte alignment** (`len % 8 == 0`) before decoding — it confirms the hypothesis in one line.
- **Learn the common magic bytes by sight:** `FF D8 FF` JPEG, `89 50 4E 47` PNG, `50 4B 03 04` ZIP, `7F 45 4C 46` ELF.
- **If `strings` finds nothing in an image, look at it.** Flags can be visual.
- **Carried forward from Autorev 1:** magic-byte recognition (`7f454c46` → ELF there, `FFD8FFE0` → JPEG here) was the deciding pivot both times.

**Security:**
- Encoding (binary text, Base64, hex) is **not** encryption — it hides nothing from anyone who recognises it.
- Data-loss-prevention and AV that only scan by extension or by known byte signatures can miss files re-encoded as text — a real exfiltration trick.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Treat the file as **bits-as-text**, not a binary | Content is only ASCII `0`/`1`; length divisible by 8 |
| 2 | Identify the decoded bytes as **JPEG** | First four bytes `FF D8 FF E0` (`JFIF`) |
| 3 | **Render** instead of grepping | No flag in `strings`/metadata; clean EOI, nothing appended |

**References:** picoCTF 2026 — Binary Digits · Tools: CyberChef (From Binary, Render Image), `file`, `exiftool`, `strings`
