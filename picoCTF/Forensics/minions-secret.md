# M1n10n'5_53cr37 — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | M1n10n'5_53cr37 | **Category** | Reverse Engineering |
| **Event** | picoMini by CMU-Africa | **Difficulty** | Medium |
| **Author** | Prince Niyonshuti N. | **Platform** | CyLab Academy |
| **Solved** | 2026-11-23 | **Artifact** | `minions.apk` |

> **Flag:** `academy{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}`
> Decoded: *"it wasn't hard unravelling the mobile code."*

## 1. TL;DR
An Android app (`.apk`) is just a ZIP. One of the app's text resources held the flag — Base32-encoded, with **two junk characters glued to the front** so a naive decode would fail. Unzip, spot the encoded string, strip the 2-char prank prefix, Base32-decode the rest.

## 2. The challenge
Minion-themed: the flag is hidden "within the Android source code," possibly in several places, and they warn they may have "pulled a fast one" (a trap/decoy). We're given `minions.apk`.

> **What's an APK?** An installable Android app — despite the extension, it's a **ZIP archive**. Unzipping gives the compiled code (`classes.dex`), resources (`res/`, `resources.arsc`), and a manifest. Any text the app uses ends up in there as data we can read.

## 3. Recon & static analysis
No `jadx`/`apktool` available, installs off — but the flag is a *string*, so `unzip` + `strings` is enough.

```bash
unzip -q minions.apk -d extracted
ls extracted
# AndroidManifest.xml  classes.dex  classes2.dex  classes3.dex  res/  resources.arsc  ...
```
Standard Kotlin app, package `com.example.picoctfimage`, no `assets/`, no native libs → flag is in code or resources.

**Sweep for the flag's shape** (a long run of only `A–Z` and `2–7` is the Base32 signature):
```bash
grep -aEro "[A-Z2-7]{30,}" extracted/
```
Surfaced one app-specific encoded string in `resources.arsc` (i.e. `res/values/strings.xml`):
```
PPMFRWCZDFNV4XWMLUL53TI43OG5PWQNDSMRPXK3TSGR3DG3BRNY4V65DIGNPW2MDCGFWDGX3DGBSDG7I=
```
The trailing `=` is Base32/Base64 padding → prime suspect.

## 4. Background: Base32
Writes arbitrary bytes using only `A–Z` and `2–7`. Reversible: `decode(encode(x)) == x`.

> **The key detail:** Base32 packs data in **blocks of 8 characters**. A valid string's length (with padding) is a multiple of 8. Our string had **81 data characters** — *impossible* for real Base32 (81 isn't a clean block length). That mismatch means something was deliberately added to break it.

## 5. Dead ends and the pivots
| # | What we tried | Result | Pivot |
|---|---|---|---|
| 1 | Decode the string as-is | "Incorrect padding" — 81 chars is invalid | Suspect a deliberate trap |
| 2 | Check `minion.jpg` for stego/appended data | Clean image, nothing after the JPEG EOI | Rule out steganography |
| 3 | Read `MainActivity` for flag-building logic | Trivial — just paints background yellow, loads image | Flag is a static resource |
| 4 | Delete each single character, re-decode | No single deletion gave readable text | More than one char is wrong — test the ends |
| 5 | Brute-force every valid-length slice, decode each | Decoding **from offset 2** gave clean text starting `academy{` | The first two chars (`PP`) are the prank. Strip them. |

> **Why brute-forcing slices works:** if you don't know where the real Base32 begins/ends, let the computer try every start/end, decode each, and keep the one that becomes readable text with `academy{`. The right alignment announces itself.

## 6. The solve
```python
#!/usr/bin/env python3
import base64
core = ("PPMFRWCZDFNV4XWMLUL53TI43OG5PWQNDSMRPXK3TSGR"
        "3DG3BRNY4V65DIGNPW2MDCGFWDGX3DGBSDG7I")
s = core[2:]                       # drop the 'PP' prank prefix
s += "=" * ((-len(s)) % 8)         # re-pad to a multiple of 8
flag = base64.b32decode(s).decode()
print(flag)
# -> academy{1t_w4sn7_h4rd_unr4v3l1n9_th3_m0b1l3_c0d3}
```

## 7. Takeaways
**Solving:**
- An APK is a ZIP — `unzip` + `strings` + `grep` go far even without a decompiler.
- Encoding fingerprints: only `A–Z2–7` → Base32; `A–Za–z0–9+/` with `=` → Base64.
- When a decoder complains about length/padding, suspect an intentional trap — trim the ends, then brute-force the alignment.
- Rule out hiding spots methodically (image, code, resources).

**Security:**
- Encoding is **not** encryption. Base32/Base64 only disguises data; anyone reverses it in seconds.
- Never ship secrets in an app's code or resources — APKs are trivially unpacked.

**References:** picoMini by CMU-Africa — M1n10n'5_53cr37 · Tools: unzip, strings, grep, Python `base64.b32decode`
