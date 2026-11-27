# Autorev 1 — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Autorev 1 | **Category** | Reverse Engineering |
| **Event** | picoCTF 2026 | **Difficulty** | Medium |
| **Author** | SkrubLawd | **Platform** | CyLab Academy |
| **Solved** | 2026-11-23 | **Service** | `nc chatelaine.cylabacademy.net <port>` |

> **Flag:** `academy{4u7o_r3v_g0_brrr_78c345aa}`
> *"auto rev go brrr"* — automated reversing, at speed.

## 1. TL;DR
The service sends **20 small programs**, one at a time, and gives **10 seconds each** to report a secret number hidden inside. Every program is the same template: it hard-codes a number, reads one from you, compares them. Reversing each by hand against a 10-second clock is hopeless, so we script extracting the hard-coded number straight from each program's bytes and reply instantly — 20/20.

## 2. The challenge
```
Welcome! ... I have 20 binaries I'm going to send you and you have
10 seconds EACH to get the secret in each one. Good luck >:)
Here's the next binary in bytes:
7f454c4602010100...        <- a whole ELF program, as hex
What's the secret?:
```
`7f454c46` = `\x7fELF` → each round is a tiny Linux program handed to us as hex. **"Autorev" + "speed" = the task is automation, not reversing any single binary.**

## 3. Reversing one sample
Grab one from a throwaway connection, convert hex → file, inspect calmly:
```bash
python3 -c "open('s.elf','wb').write(bytes.fromhex(open('hex.txt').read().strip()))"
file s.elf   # ELF 64-bit LSB executable, x86-64, dynamically linked, not stripped
```
Not stripped, and a symbol names the source `template_binary.c` → all 20 share one structure.

`objdump -d` of `main`:
```
0000000000401136 <main>:
  40113e:  mov  DWORD PTR [rbp-0x4],0xf14a7a39   ; secret = 0xf14a7a39
  401145:  mov  DWORD PTR [rbp-0x8],0x0          ; input  = 0
  401151:  call puts                             ; "What's the secret?:"
  401167:  call __isoc99_scanf                   ; read input into [rbp-8]
  40116c:  mov  eax,DWORD PTR [rbp-0x8]          ; eax = input
  40116f:  cmp  DWORD PTR [rbp-0x4],eax          ; secret == input ?
  401174:  call puts                             ; "Correct!"
  401180:  call puts                             ; "Nice try :("
```
Decompiled:
```c
int main(void) {
    unsigned int secret = 0xf14a7a39;   // the value to report
    unsigned int input  = 0;
    puts("What's the secret?:");
    scanf("%u", &input);                // %u = unsigned decimal
    if (secret == input) puts("Correct!");
    else                 puts("Nice try :(");
}
```
Answer per binary = one hardcoded 32-bit value as unsigned decimal. (`0xf14a7a39` = `4048190009`.)

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Reverse each by hand in Ghidra | Correct, but Ghidra takes tens of seconds to load one binary — you'd miss the 10s window on #1 |
| Ghidra headless / full disassembly per binary | Still ~20–40 s each; over budget |
| Run each and brute-force the input | ~4 billion possibilities — no chance in 10s |
| **Byte-signature extraction (chosen)** | Secret is the immediate in `mov [rbp-4], imm32`, template makes surrounding bytes fixed → match a 7-byte signature, read 4 bytes, instant |

> **The signature:** `c7 45 fc <imm32>` (set secret) is always immediately followed by `c7 45 f8 00 00 00 00` (zero the input). Searching `c7 45 fc (....) c7 45 f8` pins the secret; the 4 captured bytes (little-endian) are the number to send.

## 5. The auto-solver
```python
#!/usr/bin/env python3
import re, struct
from pwn import remote, context
context.log_level = "warn"

def extract_secret(blob):
    m = re.search(rb"\xc7\x45\xfc(.{4})\xc7\x45\xf8", blob, re.DOTALL)  # template-specific
    if not m:
        m = re.search(rb"\xc7\x45\xfc(.{4})", blob, re.DOTALL)          # fallback
    return struct.unpack("<I", m.group(1))[0]

io = remote("chatelaine.cylabacademy.net", 0)  # set port per instance
for i in range(20):
    while True:
        s = io.recvline().strip()
        if len(s) > 100 and all(c in b"0123456789abcdefABCDEF" for c in s):
            break
    blob = bytes.fromhex(s.decode())
    secret = extract_secret(blob)
    io.sendlineafter(b"secret?:", str(secret).encode())
    print(f"[{i:2}] secret={secret:<12} ->", io.recvline().strip().decode())

print(io.recvall(timeout=5).decode())   # flag after round 20
```
> **Gotcha:** use `re.DOTALL` — a captured secret byte can be `0x0a` (newline), which `.` won't match otherwise. A subtle bug that would fail random rounds.

## 6. The run
```
[ 0] secret=3659724637   -> Correct!
 ...
[19] secret=1182886676   -> Correct!
Woah, how'd you do that??
Here's your flag: academy{4u7o_r3v_g0_brrr_78c345aa}
```

## 7. Takeaways
**Solving:**
- When a challenge stresses **speed** or sends **many** inputs, the task is automation. Reverse *one* by hand, then script the rest.
- Match the tool to the job: a decompiler is for understanding, a byte-pattern match is for extracting at scale.
- Not-stripped binaries + source symbols (`template_binary.c`) hint all instances share one structure.

**Security:**
- A comparison against a hard-coded value is not a secret — the constant ships inside the binary and is trivially recovered.
- Client-side checks can always be read and bypassed.

**References:** picoCTF 2026 — Autorev 1 · Tools: pwntools, objdump, Python struct/re
