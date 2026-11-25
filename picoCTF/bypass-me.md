# Bypass Me — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Bypass Me | **Category** | Reverse Engineering |
| **Event** | picoCTF 2026 | **Difficulty** | Medium |
| **Author** | Prince Niyonshuti N. | **Platform** | CyLab Academy |
| **Solved** | 2026-11-23 | **Access** | `ssh ctf-player@xebec.cylabacademy.net` (SSH box) |

> **Flag:** `academy{d3bugg3r_p0w3r_is_4w3s0m3_e9ac31f3}`
> *"debugger power is awesome"* — and that was the whole intended technique.

## 1. TL;DR
A program asks for a password and prints a flag only if you get it right. Instead of guessing, attach a debugger (**LLDB**), pause at the exact moment it compares your input to the real password, and read the real password from memory. Then type it into a **normal** run — because the program needs a privilege the debugger takes away in order to open the flag file.

## 2. The challenge
SSH in, find `bypassme.bin`, a password gate. Don't brute-force — *reverse/debug* to bypass auth. Running it:
```
[3 tries left] Enter password: SuperSecure
Raw Input:      [SuperSecure]
Sanitized Input:[SuperSecure]
Hint: Input must match something special...
Access Denied
```

## 3. Background
- A **debugger** (LLDB) runs a program under control: pause at a breakpoint, inspect/modify memory and registers.
- Programs compare strings with `strcmp(a, b)` → returns `0` when equal.
- On x86-64, the first two args are in `$rdi` (1st) and `$rsi` (2nd). Stop at `strcmp` and those registers point at the two strings being compared — one is your input, the other is the real password.

## 4. Recon
Binary wasn't stripped, so symbols survive:
- `auth_sequence()` — the prompt/try loop.
- `sanitize()` — keeps only letters (the "Sanitized Input" line + "something special" hint).
- `decode_password()` — builds the real password in memory.
- `strcmp(...)` — compares sanitized input vs real password.
- Success → `fopen("../../root/flag.txt")` → prints `🎉 Flag: %s`.

> **Why not just type the password?** `sanitize()` strips characters, and the hint implies the password contains characters you can't submit intact. Read the expected password from memory instead.

## 5. Leaking the password in LLDB
```
lldb ./bypassme.bin
(lldb) b strcmp
(lldb) run
      # type any password at the prompt, then Enter
(lldb) p (char*)$rdi      # 1st argument
(lldb) p (char*)$rsi      # 2nd argument
```
**Which operand is the password?** (first pivot) Re-run and type a distinct marker so you can tell them apart:
```
(lldb) p (char*)$rdi  -> "hejefojef"      # changes with what WE type  => our input
(lldb) p (char*)$rsi  -> "SuperSecure"    # stays the same             => the PASSWORD
```
The operand that *doesn't change with your input* is the real password — here `SuperSecure`.

## 6. The pivots
| What we tried | Result | Pivot |
|---|---|---|
| Read one strcmp operand, assume it's the password | Ambiguous — first run showed the same string both sides | Re-run with a distinct marker; the unchanging operand is the password |
| Force the check: `thread return 0` on strcmp, continue | Auth passed but printed **"Flag file not found."** | The debugger itself is the problem — leak, don't patch |
| Run the real binary normally and type the leaked password | "Authenticating..." → flag printed | **The solve:** LLDB only to recover the password; the flag is read by a normal, privileged run |

> **Why the bypass printed "Flag file not found":** the flag is `/root/flag.txt`, unreadable by a normal user. The binary reads it only because it runs with elevated privilege (**setuid**). But running under a debugger, Linux's `ptrace` rules **drop that privilege** (else debuggers could read any protected file). So under LLDB the program loses the right to open the flag, even though auth succeeded. That's exactly why the challenge says *leak the password*.

## 7. Getting the flag
```
(lldb) quit
./bypassme.bin
      # Enter password:
SuperSecure
```
```
[3 tries left] Enter password: SuperSecure
...
Authenticating...
🎉 Flag: academy{d3bugg3r_p0w3r_is_4w3s0m3_e9ac31f3}
```

## 8. Takeaways
**Solving:**
- A debugger reads secrets a program holds in memory — break where the secret is *used* (the `strcmp`) and inspect the arguments.
- Know the calling convention: x86-64 passes the first two args in `$rdi`/`$rsi`.
- Disambiguate with a marker input so you can tell your data from the program's.
- `ptrace` drops setuid privileges — if a bypass works but can't read a protected file, recover the secret under the debugger and run the program normally.

**Security:**
- A password that exists in a program's memory is not secret — anyone who can run the binary can pause it and read the comparison.
- Input sanitization is not authentication.
- Real systems compare a *hash* of the input to a stored hash, so the secret never sits in memory in the clear.

**References:** picoCTF 2026 — Bypass Me · Tool: LLDB · concepts: strcmp, x86-64 calling convention, setuid & ptrace
