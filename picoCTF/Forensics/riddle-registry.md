# Riddle Registry — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Riddle Registry | **Category** | Forensics |
| **Event** | picoMini by CMU-Africa | **Difficulty** | Easy |
| **Author** | Prince Niyonshuti N. | **Platform** | CyLab Academy |
| **Solved** | 2026-11-27 | **File** | `confidential.pdf` |

> **Flag:** `academy{puzzl3d_m3tadata_f0und!_9d3cc669}`
> *"puzzled metadata found"* — the document's contents were noise; its label was the secret.

## 1. TL;DR
The PDF's visible pages are deliberately garbled filler. The flag is stored in the PDF's **metadata** (the Author field), encoded in **Base64**. Reading the metadata with `exiftool` and decoding the value gives the flag.

## 2. The challenge
> *"You've stumbled upon a peculiar PDF filled with what seems like nothing more than garbled nonsense... Amidst the chaos lies a hidden treasure... uncover the flag within the metadata."*

**The description tells you two things: the page content is a decoy ("garbled nonsense"), and the target is the metadata.** Don't spend time trying to make sense of the page text.

## 3. Analysis

### 3.1 Read the metadata, not the pages
```bash
exiftool confidential.pdf
pdfinfo  confidential.pdf
```
Most fields are ordinary (Producer, Creation Date, Page Count…). One field stands out because it isn't readable text:
```
Author : YWNhZGVteXtwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV85ZDNjYzY2OX0=
```

### 3.2 Recognise the encoding
| Clue | What it tells you |
|---|---|
| Only `A–Z a–z 0–9 + /` characters | Base64 alphabet |
| Ends with `=` | Base64 padding |
| Length 56 (a multiple of 4) | Valid Base64 length |
| Starts with `YWNhZGVteX` | That's `academy{` in Base64. **The flag prefix is recognisable while it's still encoded** |

### 3.3 Decode
```bash
echo 'YWNhZGVteXtwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV85ZDNjYzY2OX0=' | base64 -d
# academy{puzzl3d_m3tadata_f0und!_9d3cc669}
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Read the PDF in a viewer | Pages are deliberate gibberish, a decoy |
| `strings confidential.pdf \| grep academy` | Flag is Base64-encoded, so the plain text `academy` never appears |
| Look for hidden layers, white text, embedded files | Possible in PDF forensics, but the description already points to the metadata. Over-thinking an Easy challenge |
| **`exiftool` → spot the odd field → Base64 decode (chosen)** | Two commands, done |

> **The core insight:** metadata is part of the file but is never shown on the page. A field like Author is free text, so it can hold anything, including an encoded secret.

## 5. The solver

**CyberChef recipe (what we used):**
```
From Base64 (Alphabet: A-Za-z0-9+/=)
```

**Command-line equivalent:**
```bash
exiftool -Author -b confidential.pdf | base64 -d
```
> **Gotcha:** `strings | grep academy` finds nothing because the flag is encoded. Search for the **encoded prefix** instead: `grep YWNhZGVteX` (and `ZmxhZ3` for `flag{`, `cGljb0NURn` for `picoCTF{`).

## 6. The run
```
academy{puzzl3d_m3tadata_f0und!_9d3cc669}
```

## 7. Takeaways
**Solving:**
- **Read the challenge description closely.** It named both the decoy (the page text) and the target (the metadata).
- **Run `exiftool` early on any file.** Metadata is the cheapest place to check.
- **Recognise Base64 by sight:** its character set, `=` padding, and a length that's a multiple of 4.
- **Memorise your flag prefix in Base64** (`academy{` → `YWNhZGVteX`) so you can grep for it while it's still encoded.
- **Carried forward:** *read the metadata for intent* (Autorev 1's symbols, Binary Digits' file type). Here the metadata was the whole answer.

**Security:**
- Document metadata (author, software, paths, edit history) leaks information about who made a file and how. Organisations should strip it before sharing documents (`exiftool -all= file.pdf`).
- Base64 is an encoding, not encryption. Anyone can decode it.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Skip the page text and go to the **metadata** | The description called the content "garbled nonsense" and named the metadata |
| 2 | Spot the **odd field** | The Author value was a long random-looking string, not a name |
| 3 | **Base64-decode** it | Base64 character set, `=` padding, `YWNhZGVteX` prefix |

**References:** picoMini by CMU-Africa — Riddle Registry · Tools: exiftool, pdfinfo, CyberChef (From Base64)
