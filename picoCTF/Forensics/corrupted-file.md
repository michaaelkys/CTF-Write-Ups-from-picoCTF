# Corrupted file — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Corrupted file | **Category** | Forensics |
| **Event** | picoMini by CMU-Africa | **Difficulty** | Easy |
| **Author** | Yahaya Meddy | **Platform** | CyLab Academy |
| **Solved** | 2026-11-27 | **File** | `file` (no extension) |

> **Flag:** `academy{r3st0r1ng_th3_by73s_4efc9f4e}`
> *"restoring the bytes"*: two bytes stood between a broken file and a readable image.

## 1. TL;DR
`file` is a **JPEG whose first two bytes were overwritten**. Its signature should start `FF D8`, but it starts `5C 78` (the ASCII text `\x`). Everything after that is intact, including the `FF E0` marker and the `JFIF` text. Writing `FF D8` back at offset 0 repairs the file, and the flag is drawn in the image.

## 2. The challenge
> *"This file seems broken... or is it? Maybe a couple of bytes could make all the difference. Can you figure out how to bring it back to life?"*

**"A couple of bytes" is a literal hint.** The damage is small and in one place, so the task is to repair the file, not dig through it.

## 3. Analysis

### 3.1 Look at the first bytes
```bash
xxd file | head -n 3
```
```
00000000: 5c78 ffe0 0010 4a46 4946 0001 0100 0001  \x....JFIF......
00000010: 0001 0000 ffdb 0043 0008 0606 0706 0508  .......C........
00000020: 0707 0709 0908 0a0c 140d 0c0b 0b0c 1912  ................
```

### 3.2 Read the dump
| Offset | Bytes | Meaning |
|---|---|---|
| `0x00` | `5C 78` | ASCII `\x`. **Not a valid signature. This is the damage** |
| `0x02` | `FF E0` | JPEG **APP0** marker (the header block that holds JFIF) ✔ |
| `0x04` | `00 10` | APP0 length = 16 bytes ✔ |
| `0x06` | `4A 46 49 46 00` | `JFIF\0`, the JPEG/JFIF identifier ✔ |
| `0x14` | `FF DB` | JPEG quantization table marker ✔ |

Everything from offset 2 onwards is a valid JPEG. Only the **2-byte Start-Of-Image marker** is wrong.

### 3.3 Compare with a known-good JPEG
```
JPEG should start:  FF D8 FF E0 00 10 4A 46 49 46   (seen in Binary Digits)
This file starts:   5C 78 FF E0 00 10 4A 46 49 46
                    ^^^^^ only difference
```
`5C 78` is the text `\x`, so it looks like someone wrote the escape-sequence text instead of the real bytes. Without the correct `FF D8`, `file` and image viewers don't recognise it as a JPEG and treat it as broken.

### 3.4 Repair
```bash
cp file fixed.jpg                                          # always work on a copy
printf '\xff\xd8' | dd of=fixed.jpg bs=1 seek=0 conv=notrunc
```
- `printf '\xff\xd8'` creates the two real bytes `FF D8`.
- `seek=0` writes them at the start of the file.
- `conv=notrunc` overwrites only those 2 bytes and keeps the rest of the file.

### 3.5 Verify
```bash
xxd fixed.jpg | head -n 1    # ffd8 ffe0 0010 4a46 4946 ...
file fixed.jpg               # JPEG image data, JFIF standard 1.01 ...
xdg-open fixed.jpg
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Open `file` in an image viewer | No valid signature, so it's rejected as corrupt |
| `strings file \| grep academy` | Flag is drawn in the pixels, not stored as text |
| `binwalk` / carving for embedded files | Nothing is hidden. The file itself is damaged, not concealing something |
| Rebuild the whole header by hand | Unnecessary. Only 2 bytes differ from a valid JPEG |
| **Find the file type from its surviving markers, patch the 2-byte signature (chosen)** | `FF E0` + `JFIF` identify a JPEG, and `FF D8` at offset 0 fixes it |

> **The core insight:** corruption is rarely total. The markers that survive (`FF E0`, `JFIF`, `FF DB`) tell you what the file is supposed to be, and comparing against a known-good signature shows exactly which bytes to restore.

## 5. The solver
**Command line (what we used):**
```bash
cp file fixed.jpg
printf '\xff\xd8' | dd of=fixed.jpg bs=1 seek=0 conv=notrunc
xdg-open fixed.jpg
```

**Alternatives:**
```
hexedit fixed.jpg               → overwrite 5C 78 with FF D8, Ctrl+X to save
CyberChef: To Hex → edit first two bytes to ff d8 → From Hex → Render Image
```
> **Gotcha:** without `conv=notrunc`, `dd` **truncates** the output file after the bytes it writes. You'd end up with a 2-byte file. And always patch a **copy**, so you can start again if a write goes wrong.

## 6. The run
```
$ file fixed.jpg
fixed.jpg: JPEG image data, JFIF standard 1.01 ...
```
Rendered image (white background, red text):
```
academy{r3st0r1ng_th3_by73s_4efc9f4e}
```

## 7. Takeaways
**Solving:**
- **If a file "won't open", check its signature first.** `xxd file | head -n 1` usually shows the problem.
- **Use the markers that survive to identify the file:** `JFIF` / `FF E0` → JPEG, `IHDR` → PNG, `%PDF` → PDF.
- **Know the core signatures by heart:** JPEG `FF D8 FF`, PNG `89 50 4E 47 0D 0A 1A 0A`, ZIP `50 4B 03 04`, PDF `25 50 44 46`, ELF `7F 45 4C 46`.
- **Patch in place with `dd conv=notrunc`**, and always on a copy.
- **Carried forward:**
  - *Binary Digits* gave us the reference signature `FF D8 FF E0`. This challenge is the same JPEG header, damaged.
  - *Binary Digits / Flag in Flame*: the flag was **drawn in the image** again, so you have to view it, not search it.

**Security:**
- Tools that trust the first few bytes can be fooled in both directions. A file with a broken signature can slip past type-based filters, and a fake signature can make a malicious file look harmless. Check the whole structure, not just the first bytes.
- In incident response, a "corrupted" file isn't necessarily useless. A small header repair often makes it readable again.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Treat it as a **damaged file**, not a hiding place | Description says "a couple of bytes" and "bring it back to life" |
| 2 | Identify it as a **JPEG** from the surviving markers | `FF E0` at offset 2 and `JFIF` at offset 6 |
| 3 | Find the exact damage by **comparing with a known-good signature** | `5C 78` (`\x`) where `FF D8` should be |
| 4 | **Patch the 2 bytes in place**, then view the image | `dd conv=notrunc` restored a valid JPEG and the flag was drawn in it |

## Appendix: File Signature Cheat Sheet
Use this to identify a file from its first bytes (hex), from the same bytes as ASCII, or while it's still Base64-encoded. *Offset* is where the signature starts (0 unless noted).

### Images
| Type | Hex signature | ASCII | Base64 prefix | Notes |
|---|---|---|---|---|
| **JPEG** | `FF D8 FF E0` / `FF D8 FF E1` | `ÿØÿà` + `JFIF` / `Exif` | `/9j/` | Ends `FF D9`. `E1` = camera photo with EXIF |
| **PNG** | `89 50 4E 47 0D 0A 1A 0A` | `.PNG....` | `iVBORw0KGgo` | Chunks `IHDR`…`IEND`. Ends `49 45 4E 44 AE 42 60 82` |
| **GIF** | `47 49 46 38 39 61` | `GIF89a` / `GIF87a` | `R0lGOD` | Ends `00 3B` |
| **BMP** | `42 4D` | `BM` | `Qk` | Next 4 bytes = file size (little-endian) |
| **TIFF** | `49 49 2A 00` / `4D 4D 00 2A` | `II*.` / `MM.*` | | `II` = little-endian, `MM` = big-endian |
| **WEBP** | `52 49 46 46 ?? ?? ?? ?? 57 45 42 50` | `RIFF....WEBP` | | Same RIFF container as WAV |

### Documents
| Type | Hex signature | ASCII | Base64 prefix | Notes |
|---|---|---|---|---|
| **PDF** | `25 50 44 46 2D` | `%PDF-` | `JVBERi0` | Ends `%%EOF`. Check metadata (Riddle Registry) |
| **DOCX / XLSX / PPTX** | `50 4B 03 04` | `PK..` | `UEsDB` | Really ZIPs. Unzip and look in `word/`, `xl/`, `ppt/` |
| **Old DOC / XLS / MSG** | `D0 CF 11 E0 A1 B1 1A E1` | `ÐÏ.à¡±.á` | `0M8R4K` | OLE compound file (old Office format) |
| **RTF** | `7B 5C 72 74 66` | `{\rtf` | | Plain text with markup |

### Archives & compression
| Type | Hex signature | ASCII | Base64 prefix | Notes |
|---|---|---|---|---|
| **ZIP** (also JAR, APK) | `50 4B 03 04` | `PK..` | `UEsDB` | End-of-archive record `50 4B 05 06` |
| **RAR** | `52 61 72 21 1A 07` | `Rar!..` | `UmFyIRoH` | |
| **7-Zip** | `37 7A BC AF 27 1C` | `7z¼¯'.` | `N3q8ryc` | |
| **GZIP** (`.gz`, `.tgz`) | `1F 8B` | `..` | `H4sI` | Often wraps a TAR |
| **BZIP2** | `42 5A 68` | `BZh` | `Qlpo` | |
| **XZ** | `FD 37 7A 58 5A 00` | `ý7zXZ.` | `/Td6WFo` | |
| **TAR** | `75 73 74 61 72` at offset **257** | `ustar` | | No magic at offset 0 |

### Executables & code
| Type | Hex signature | ASCII | Base64 prefix | Notes |
|---|---|---|---|---|
| **ELF** (Linux) | `7F 45 4C 46` | `.ELF` | `f0VMRg` | Autorev 1 |
| **PE / EXE / DLL** (Windows) | `4D 5A` | `MZ` | `TVq` | `PE\0\0` appears later in the header |
| **Mach-O** (macOS) | `CF FA ED FE` / `FE ED FA CF` | | | 64-bit, little- / big-endian |
| **Java class** | `CA FE BA BE` | `Êþº¾` | `yv66vg` | |
| **Shell / Python script** | `23 21` | `#!` | `IyE` | "Shebang" line, e.g. `#!/bin/bash` |

### Network & data
| Type | Hex signature | ASCII | Base64 prefix | Notes |
|---|---|---|---|---|
| **PCAP** | `D4 C3 B2 A1` / `A1 B2 C3 D4` | | `1MOyoQ` | Open in Wireshark |
| **PCAPNG** | `0A 0D 0D 0A` | | | Newer Wireshark format |
| **SQLite DB** | `53 51 4C 69 74 65 20 66 6F 72 6D 61 74 20 33 00` | `SQLite format 3` | `U1FMaXRl` | Open with `sqlite3` |

### Audio & video
| Type | Hex signature | ASCII | Notes |
|---|---|---|---|
| **WAV** | `52 49 46 46 ?? ?? ?? ?? 57 41 56 45` | `RIFF....WAVE` | Often used for audio stego or spectrogram images |
| **MP3** | `49 44 33` / `FF FB` | `ID3` | `ID3` = has a metadata tag |
| **OGG** | `4F 67 67 53` | `OggS` | |
| **FLAC** | `66 4C 61 43` | `fLaC` | |
| **MP4 / MOV** | `66 74 79 70` at offset **4** | `....ftyp` | |

### Text encodings (not files, but you'll meet them constantly)
| Encoding | How to spot it | `academy{` looks like |
|---|---|---|
| **Binary** | Only `0`/`1`, length a multiple of 8 | `01100001 01100011 …` (Binary Digits) |
| **Hex** | Only `0–9A–F`, even length | `61 63 61 64 65 6D 79 7B` (Flag in Flame) |
| **Base64** | `A–Z a–z 0–9 + /`, `=` padding, length a multiple of 4 | `YWNhZGVteX` (Riddle Registry) |
| **Base32** | `A–Z 2–7` only, `=` padding | `MFRWCZDFNV4XW` |
| **URL encoding** | `%XX` sequences | `%61%63%61…` |

> **How to use it:** run `xxd file | head -n 2` and match the first bytes here. If the extension disagrees with the signature, **trust the signature**. If nothing matches, the file is probably still encoded (check the text encodings table) or damaged (compare the parts that survived, like this challenge).

**References:** picoMini by CMU-Africa: Corrupted file · Tools: `xxd`, `dd`, `printf`, `file`, `hexedit`, CyberChef
