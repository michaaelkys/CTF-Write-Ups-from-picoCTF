# Verify — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Verify | **Category** | Forensics |
| **Event** | picoCTF 2024 | **Difficulty** | Easy |
| **Author** | Jeffery John | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **Access** | `ssh -p <port> ctf-player@chatelaine.cylabacademy.net` |

> **Flag:** `academy{trust_but_verify_a906870d}`
> *"trust but verify"*. One real file among ~300 imitations, and the hash tells you which.

## 1. TL;DR
The SSH box holds a `files/` folder with **~300 encrypted files** and a `decrypt.sh` script. Only one file is the real flag. We were given its **SHA-256 checksum**. Hashing every file with `sha256sum` and grepping for that checksum finds the right one (`a906870d`) in a single command. Decrypting it gives the flag.

## 2. The challenge
> *"People keep trying to trick my players with imitation flags... I'm going to provide the SHA-256 hash and a decrypt script to help you know that my flags are legitimate."*

```
Checksum: deb0a6ad170cd3033d30cae15146719dd4abf37d39136bcb10d88d9c33413374
Decrypt:  ./decrypt.sh files/<file>
```

## 3. Analysis

### 3.1 Recon
```bash
ssh -p <port> ctf-player@chatelaine.cylabacademy.net
ls          # decrypt.sh  files/
ls files/   # ~300 files with random 8-character names
```
Too many to check by hand, and decrypting each one could produce **imitation flags**, which is exactly what the description warns about.

### 3.2 What a SHA-256 checksum is
A **hash** is a fingerprint of a file's exact contents. Change one byte and the hash changes completely. SHA-256 always gives 64 hex characters. If a file's hash matches the published checksum, it's the genuine file.

### 3.3 Hash everything, keep the match
```bash
sha256sum files/* | grep deb0a6ad170cd3033d30cae15146719dd4abf37d39136bcb10d88d9c33413374
```
```
deb0a6ad170cd3033d30cae15146719dd4abf37d39136bcb10d88d9c33413374  files/a906870d
```
Exactly one match: **`files/a906870d`**.

### 3.4 Decrypt only the verified file
```bash
./decrypt.sh files/a906870d
# academy{trust_but_verify_a906870d}
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| Decrypt files one by one until a flag appears | ~300 runs, and the decoys produce **imitation flags**, so you can't tell real from fake |
| Decrypt all files in a loop and grep for `academy{` | Gives you a pile of flag-shaped strings, the trap the challenge sets |
| `sha256sum` each file manually | Correct but slow |
| **`sha256sum files/* \| grep <checksum>` (chosen)** | Hashes every file in one pass and shows only the verified one |

> **The core insight:** verify first, then trust. The checksum isn't a hint about the content. It's a way to identify the right file without opening any of them.

## 5. The solver
**Command line (what we used):**
```bash
sha256sum files/* | grep deb0a6ad170cd3033d30cae15146719dd4abf37d39136bcb10d88d9c33413374
./decrypt.sh files/a906870d
```
> **Gotcha:** paste the **full** 64-character checksum into `grep`. A partial hash can match more than one file by chance. And `cat decrypt.sh` before running it: it's a plain-text script, so reading it is safe and shows what it does.

## 6. The run
```
$ sha256sum files/* | grep deb0a6ad...3413374
deb0a6ad170cd3033d30cae15146719dd4abf37d39136bcb10d88d9c33413374  files/a906870d
$ ./decrypt.sh files/a906870d
academy{trust_but_verify_a906870d}
```

## 7. Takeaways
**Solving:**
- **When there are many files and one known fingerprint, hash them all and filter.** Don't open them one by one.
- **Pipe-and-filter is your friend:** `command | grep <pattern>` turns hundreds of lines into one.
- **Read the description for the trap:** "imitation flags" meant decrypting everything would mislead you.
- **Carried forward:**
  - *DISKO 1*: cheapest check first. One `sha256sum | grep` line instead of 300 decryptions.
  - *Autorev 1*: when there are many inputs, use the shell to do the repetitive work.

**Security (SOC angle):**
- **Hashes are how defenders verify files.** Downloads publish SHA-256 sums so you can confirm you got the genuine file, and SOC teams use file hashes as **indicators of compromise (IOCs)** to find known malware across many machines.
- `sha256sum <file>` and searching the hash on VirusTotal is a standard first triage step for a suspicious file.
- Hashes prove **integrity**, not safety. A file can match its published hash and still be malicious if the publisher is.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Don't decrypt blindly, **verify first** | ~300 files + the warning about "imitation flags" |
| 2 | **Hash all files at once** and grep for the checksum | Known SHA-256 checksum given in the description |
| 3 | Decrypt **only** the matching file | Exactly one match: `files/a906870d` |

**References:** picoCTF 2024: Verify · Tools: `ssh`, `sha256sum`, `grep`, `decrypt.sh`
