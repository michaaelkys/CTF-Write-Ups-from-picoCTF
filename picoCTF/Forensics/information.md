# information — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | information | **Category** | Forensics |
| **Event** | picoCTF 2021 | **Difficulty** | Easy |
| **Author** | susie | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **File** | `cat.jpg` (2560×1598, 878 kB) |

> **Flag:** `academy{the_m3tadata_1s_modified}`
> *"the metadata is modified"*. The flag is right there in the file's info, Base64-encoded.

## 1. TL;DR
`cat.jpg` is a normal photo of a cat. Its metadata has a **`License`** field containing a **Base64** string instead of a licence name. Decoding it gives the flag. The `XMP Toolkit: ExifTool` entry shows the metadata was edited on purpose.

## 2. The challenge
> *"Files can always be changed in a secret way. Can you find the flag?"*

"Changed in a secret way" points at **metadata**: data stored in the file that isn't part of the visible image.

## 3. Analysis

### 3.1 Cheapest check: metadata
```bash
exiftool cat.jpg
```
```
Copyright Notice : PicoCTF
XMP Toolkit      : Image::ExifTool 13.25
License          : YWNhZGVteXt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9
Rights           : PicoCTF
Image Size       : 2560x1598
```
| Clue | Meaning |
|---|---|
| **`License`** holds random-looking text | A licence field should name a licence, not a blob |
| `XMP Toolkit : Image::ExifTool` | The metadata was **edited with ExifTool**, i.e. added deliberately |
| Starts `YWNhZGVteX`, no `=` padding needed here | `academy{` in **Base64** |

### 3.2 Decode
```bash
echo 'YWNhZGVteXt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9' | base64 -d
# academy{the_m3tadata_1s_modified}
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Look at the cat photo | Nothing hidden in the visible image |
| LSB / stego tools | Over-thinking it. "Changed in a secret way" = metadata, not pixels |
| `strings \| grep academy` | Flag is Base64-encoded, so plain `academy` never appears |
| **`exiftool` → odd field → Base64 decode (chosen)** | Two commands |

> **The core insight:** same pattern as Riddle Registry and CanYouSee. A metadata field whose value doesn't match its name is where the flag hides.

## 5. The solver
**Command line (what we used):**
```bash
exiftool cat.jpg
echo 'YWNhZGVteXt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9' | base64 -d
```
**One-liner:**
```bash
exiftool -s3 -License cat.jpg | base64 -d
```
> **Gotcha:** the flag sat in `License`, but it could as easily be in `Copyright`, `Comment`, `Artist`, `UserComment` or any editable field. Scan the **whole** `exiftool` output for anything that looks encoded, don't assume a specific field.

## 6. The run
```
$ exiftool -s3 -License cat.jpg | base64 -d
academy{the_m3tadata_1s_modified}
```

## 7. Takeaways
**Solving:**
- **`exiftool` first on any image or document.** This is the fourth challenge it cracked (Riddle Registry, RED, CanYouSee, information).
- **Scan every metadata field,** not just the obvious ones. The flag can hide in `License`, `Copyright`, `Comment`, etc.
- **`XMP Toolkit: ExifTool` is a tell** that someone modified the metadata.
- **Carried forward:**
  - *CanYouSee*: identical method (JPEG + Base64 in a metadata field), different field name.
  - The Base64-vs-hex test from RED/CanYouSee: lower-case letters → Base64.

**Security:**
- Metadata travels with files silently. It can leak author names, software, GPS location and more. Strip it before sharing: `exiftool -all= cat.jpg`.
- Editable metadata fields are also a quiet channel to **plant data** in an otherwise innocent-looking file.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Check **metadata**, not the image | "Changed in a secret way" |
| 2 | Spot the **`License`** field | Its value is a blob, not a licence; `XMP Toolkit: ExifTool` shows it was edited |
| 3 | **Base64-decode** it | `YWNhZGVteX` prefix |

**References:** picoCTF 2021: information · Tools: `exiftool`, `base64`
