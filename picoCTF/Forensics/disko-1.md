# DISKO 1 — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | DISKO 1 | **Category** | Forensics |
| **Event** | picoCTF | **Difficulty** | Easy |
| **Author** | Darkraicg492 | **Platform** | CyLab Academy |
| **Solved** | 2026-11-27 | **File** | `disko-1.dd.gz` |

> **Flag:** `academy{1t5_ju5t_4_5tr1n9_3f20e35d}`
> *"it's just a string"*: no mounting or file carving needed. The flag was sitting in the raw disk as plain text.

## 1. TL;DR
`disko-1.dd.gz` is a **gzip-compressed raw disk image** of a **FAT filesystem** (OEM-ID `mkfs.fat`). After decompressing it, running `strings` over the whole image and grepping for the flag prefix finds the flag as plain text. A raw image holds every byte of the disk, so any text stored in it can be found without mounting the filesystem.

## 2. The challenge
> *"Can you find the flag in this disk image?"*

The filename gives you the layers: **`.gz`** means gzip-compressed, and **`.dd`** means a raw, byte-for-byte disk copy (named after the `dd` tool used to make it).

## 3. Analysis

### 3.1 Peel the compression
```bash
file disko-1.dd.gz          # gzip compressed data, was "disko-1.dd" ...
gunzip -k disko-1.dd.gz     # -k keeps the original
```
Matches the cheat sheet from Corrupted file: GZIP = `1F 8B`.

### 3.2 Identify the disk
```bash
file disko-1.dd
# DOS/MBR boot sector, ... OEM-ID "mkfs.fat", ... FAT ...
```
| Field | Meaning |
|---|---|
| `DOS/MBR boot sector` | The first sector is a boot sector, so this is a disk/partition image |
| `OEM-ID "mkfs.fat"` | A label in the boot sector: the disk was formatted on Linux with `mkfs.fat` |
| `FAT` | The filesystem is FAT, the same simple format used on USB sticks and SD cards |

So the image holds a real filesystem with files and folders.

### 3.3 Search the raw image
```bash
strings disko-1.dd | grep -i "academy"
# academy{1t5_ju5t_4_5tr1n9_3f20e35d}
```
`strings` reads **every byte** of the image: file contents, free space, deleted data, everything. It doesn't need to understand the filesystem.

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| `cat disko-1.dd` | Dumps raw binary into the terminal. Control bytes get treated as escape codes and the terminal spams replies like `1;2c`. Useless, and it messes up your terminal |
| Mount the image (`mount -o loop,ro`) and browse | Works, but needs `sudo`, and you'd still have to open files one by one |
| `fls` / `icat` (Sleuth Kit) to list and extract files | The proper forensic method, but more than an Easy challenge needs |
| **`strings` + `grep` on the raw image (chosen)** | One command. Finds the flag wherever it is on the disk |

> **The core insight:** a raw disk image is just bytes. Before mounting or carving anything, try the cheapest check: is the flag in there as plain text? Here the flag name says it outright: *it's just a string*.

## 5. The solver
**Command line (what we used):**
```bash
gunzip -k disko-1.dd.gz
file disko-1.dd
strings disko-1.dd | grep -i "academy"
```
**One-liner without decompressing to disk:**
```bash
zcat disko-1.dd.gz | strings | grep -i "academy"
```
> **Gotcha:** **never `cat` a binary file.** We hit this: `cat` on the image filled the prompt with `1;2c1;2c1;2c…`. That's the terminal replying to "what terminal are you?" control codes it found in the random bytes. Fix it with `reset`, and use `xxd`, `strings` or `file` to look at binaries.

## 6. The run
```
$ strings disko-1.dd | grep -i "academy"
academy{1t5_ju5t_4_5tr1n9_3f20e35d}
```

## 7. Takeaways
**Solving:**
- **Read the extensions as layers:** `.dd.gz` = compression, then a raw disk. Peel them one at a time and run `file` after each.
- **Try the cheap check first.** `strings | grep <prefix>` on a whole disk image takes seconds. Escalate to mounting, `fls`/`icat` or carving only if it fails.
- **`file` output is evidence:** `DOS/MBR boot sector` + `OEM-ID "mkfs.fat"` = a FAT disk image formatted on Linux.
- **`cat` is for text only.** Use `xxd`, `strings` and `file` for binaries.
- **Carried forward:**
  - *Corrupted file* cheat sheet: `.gz` = GZIP (`1F 8B`), decompress first.
  - *Riddle Registry*: if `grep academy` had found nothing, the next step would be grepping for **encoded** prefixes (`YWNhZGVteX`, hex `6163616465`).

**Security / DFIR:**
- A raw image (`dd`) is the standard forensic copy. It keeps **everything**, including deleted files and free space, which is why investigators image a disk instead of copying its files.
- Always work on images **read-only** (`mount -o ro`), never on the original evidence.
- Deleting a file doesn't wipe its bytes. Its text can still be found with `strings` until that space is overwritten.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | **Decompress** before analysing | `.gz` extension / GZIP signature |
| 2 | Stop using `cat`, switch to binary-safe tools | Terminal filled with `1;2c` escape-code replies |
| 3 | Identify the image as a **FAT filesystem** | `file` showed `DOS/MBR boot sector`, `OEM-ID "mkfs.fat"` |
| 4 | Search the **raw bytes** instead of mounting | Easy challenge, so try the cheapest check first: `strings \| grep` found the flag |

**References:** picoCTF: DISKO 1 · Tools: `gunzip`, `file`, `strings`, `grep` · Further: Sleuth Kit (`fls`, `icat`), `mount -o loop,ro`
