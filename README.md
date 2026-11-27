# CTF Write-Ups — picoCTF

Step-by-step write-ups of the picoCTF / picoMini challenges I've solved on **CyLab Academy**. Each write-up shows the method that worked **and the pivots along the way**: what I tried, what the evidence showed, and why I changed direction.

## About me
BSIT Cybersecurity student working toward a **SOC Analyst** role. I'm using CTFs to build practical skills in forensics, reverse engineering and incident analysis: recognising what an artifact is, peeling back layers of encoding, and turning evidence into answers.

## Progress
**10 challenges solved** · 5 Forensics · 5 Reverse Engineering

### Forensics
| Challenge | Event | Difficulty | Key skill | Write-up | PDF |
|---|---|---|---|---|---|
| Binary Digits | picoCTF 2026 | Easy | Bits-as-text → bytes, JPEG magic bytes | [md](picoCTF/Reverse%20Engineering/binary-digits.md) | [pdf](picoCTF/PDFs/BinaryDigits.pdf) |
| Riddle Registry | picoMini (CMU-Africa) | Easy | PDF metadata, Base64 | [md](picoCTF/Forensics/riddle-registry.md) | [pdf](picoCTF/PDFs/RiddleRegistry.pdf) |
| Flag in Flame | picoMini (CMU-Africa) | Easy | Layered encoding: Base64 → PNG → hex | [md](picoCTF/Forensics/flag-in-flame.md) | [pdf](picoCTF/PDFs/FlagInFlame.pdf) |
| Corrupted file | picoMini (CMU-Africa) | Easy | File-header repair, signature cheat sheet | [md](picoCTF/Forensics/corrupted-file.md) | [pdf](picoCTF/PDFs/CorruptedFile.pdf) |
| DISKO 1 | picoCTF | Easy | Disk image triage, `strings` | [md](picoCTF/Forensics/disko-1.md) | [pdf](picoCTF/PDFs/Disko1.pdf) |

### Reverse Engineering
| Challenge | Event | Difficulty | Key skill | Write-up | PDF |
|---|---|---|---|---|---|
| Autorev 1 | picoCTF 2026 | Medium | Automated ELF reversing, byte-signature extraction | [md](picoCTF/Reverse%20Engineering/autorev-1.md) | [pdf](picoCTF/PDFs/Autorev1.pdf) |
| Bypass Me | picoCTF 2026 | Medium | Debugging with LLDB, reading secrets from memory | [md](picoCTF/Reverse%20Engineering/bypass-me.md) | [pdf](picoCTF/PDFs/BypassMe.pdf) |
| Chronohack | picoCTF 2025 | Medium | Predictable time-seeded PRNG | [md](picoCTF/Reverse%20Engineering/chronohack.md) | [pdf](picoCTF/PDFs/Chronohack.pdf) |
| M1n10n'5_53cr37 | picoMini (CMU-Africa) | Medium | APK unpacking, Base32 | [md](picoCTF/Forensics/minions-secret.md) | [pdf](picoCTF/PDFs/Minions_Secret.pdf) |
| Pico Bank | picoMini (CMU-Africa) | Medium | APK decompiling, client-side auth abuse | [md](picoCTF/Forensics/pico-bank.md) | [pdf](picoCTF/PDFs/PicoBank.pdf) |

## Lessons carried forward
Each challenge builds on the ones before it:
1. **Trust the bytes, not the extension.** Check the first bytes with `xxd file | head` (Autorev 1, Binary Digits).
2. **Read metadata and symbols for intent** (Autorev 1, Riddle Registry).
3. **Recognise encodings on sight:** binary, hex, Base64, Base32, including the flag prefix while it's still encoded (`academy{` = `YWNhZGVteX` in Base64).
4. **Layers are normal. Keep peeling:** identify → decode → identify again (Flag in Flame).
5. **If an image has no text to find, look at it.** Flags can be drawn in the pixels (Binary Digits, Flag in Flame, Corrupted file).
6. **Corrupted ≠ lost.** The parts of a header that survive show which bytes to fix (Corrupted file).
7. **Try the cheapest check first.** `strings | grep` on a disk or image before mounting or carving (DISKO 1).
8. **`cat` is for text only.** Use `xxd`, `strings` and `file` for binaries.
9. **Script only when it's needed,** e.g. many repetitive rounds under a time limit (Autorev 1).

A full **file signature cheat sheet** (hex, ASCII and Base64 prefixes) is in the [Corrupted file write-up](picoCTF/Forensics/corrupted-file.md#appendix-file-signature-cheat-sheet).

## Toolkit
Kali Linux (VirtualBox) · CyberChef · `xxd` / `hexdump` / `hexedit` · `file` · `strings` · `exiftool` · `binwalk` · `dd` · Ghidra · GDB / LLDB · objdump · Python (pwntools) · Wireshark · jadx / apktool

## Write-up format
Every write-up follows the same structure:
**TL;DR → Challenge → Analysis → Key decision (approaches compared) → Solver → Run → Takeaways (solving + security) → Pivot Log**

---
*All challenges were solved in authorised CTF environments for learning purposes.*
