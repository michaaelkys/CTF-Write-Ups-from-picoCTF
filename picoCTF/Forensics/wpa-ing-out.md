# WPA-ing Out — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | WPA-ing Out | **Category** | Forensics |
| **Event** | picoCTF | **Difficulty** | Easy |
| **Author** | MistressVampy | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **File** | `wpa-ing_out.pcap` |

> **Flag:** `academy{mickeymouse}`
> *"passed over the air can be cracked"*. A reused, weak Wi-Fi password falls to a wordlist in seconds.

## 1. TL;DR
The PCAP contains a captured **WPA 4-way handshake**. Because the Wi-Fi password is a common one in **`rockyou.txt`**, `aircrack-ng` cracks it by testing every word in the list against the handshake. The recovered password (`mickeymouse`) is the flag.

## 2. The challenge
> *"...passwords passed over the AIR can be CRACKED, especially if I used the same wireless network password as one in the rockyou.txt credential dump."*

The description is the walkthrough: a Wi-Fi handshake in the PCAP, cracked with the `rockyou` wordlist. Flag format `academy{XXXXXX}`.

## 3. Analysis

### 3.1 What a WPA handshake is
When a device joins a WPA/WPA2 Wi-Fi network, it performs a **4-way handshake** with the router. This exchange proves both sides know the password **without sending it**, but it does include enough derived data that, if captured, an attacker can test password guesses against it offline. No password guess is ever sent over the air; it's all computed locally.

### 3.2 Confirm the handshake is captured
Open the PCAP in Wireshark and filter:
```
eapol
```
`EAPOL` is the protocol of the handshake. You need all **4 messages** (Msg 1/4 → 4/4). Without a complete handshake, there's nothing to crack. (SSID, the network name, is visible in the beacon frames: `wlan.fc.type_subtype == 0x08`.)

### 3.3 Crack against rockyou
```bash
# rockyou often ships gzipped
sudo gunzip /usr/share/wordlists/rockyou.txt.gz   # if needed

aircrack-ng wpa-ing_out.pcap -w /usr/share/wordlists/rockyou.txt
```
`aircrack-ng` reads the handshake from the capture, then derives a key from each wordlist entry and compares it to the captured handshake.

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| `strings` / grep the PCAP for the password | The password is never transmitted. Only the handshake is, so there's no plaintext to find |
| Brute-force all possible passwords | WPA keys are huge. Unguided brute force is hopeless |
| Capture more traffic / deauth | Pointless. The handshake is already in the provided file |
| **Dictionary attack on the handshake with `rockyou` (chosen)** | The description says the password is in `rockyou`, so a wordlist attack finds it in seconds |

> **The core insight:** you can't read a WPA password off the wire, but a captured handshake lets you **test guesses offline**. When the password is a common one, a wordlist turns "uncrackable" into seconds.

## 5. The solver
**Command line (what we used):**
```bash
aircrack-ng wpa-ing_out.pcap -w /usr/share/wordlists/rockyou.txt
```
**hashcat alternative:**
```bash
hcxpcapngtool -o hash.hc22000 wpa-ing_out.pcap   # convert the handshake
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt
```
> **Gotcha:** if `aircrack-ng` says "No valid WPA handshakes found", the capture is missing one or more of the 4 EAPOL messages. Check with the `eapol` filter in Wireshark first. Also make sure `rockyou.txt` is decompressed, aircrack can't read the `.gz` directly.

## 6. The run
```
                          KEY FOUND! [ mickeymouse ]

      Master Key : 61 64 B9 5E FC 6F 41 70 70 81 F6 40 80 9F AF B1 ...
```
Flag: `academy{mickeymouse}`

## 7. Takeaways
**Solving:**
- **A WPA PCAP + a wordlist mention = offline handshake cracking.** The tool is `aircrack-ng` or `hashcat`.
- **Confirm the handshake first** (`eapol` filter, 4 messages) before spending time cracking.
- **`rockyou.txt` is the default wordlist** for CTFs: ~14 million real leaked passwords. If a password is weak or reused, it's probably in there.
- **Carried forward:**
  - *Ph4nt0m 1ntrud3r*: a Wireshark display filter narrows millions of packets to the few that matter.
  - *Verify*: a wordlist/hash match replaces blind guessing with a systematic check.

**Security (SOC angle):**
- **This is why Wi-Fi password strength matters.** The 4-way handshake can be captured by anyone nearby, and a weak password falls to an offline wordlist attack. The computation happens on the attacker's machine, so there's nothing to detect on the network.
- Defences: a long, random passphrase (not a `rockyou` word), **WPA3** (which resists offline dictionary attacks via SAE), and enterprise auth (802.1X) instead of a shared password.
- `rockyou.txt` comes from a real 2009 breach. Checking passwords against known breach lists is standard practice now (e.g. "Have I Been Pwned").

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Recognise it as a **WPA handshake crack** | "passwords passed over the air" + "rockyou.txt" in the description |
| 2 | **Confirm the handshake** with the `eapol` filter | Cracking needs all 4 EAPOL messages present |
| 3 | **Dictionary attack** with `aircrack-ng` + rockyou | Description says the password is reused from rockyou |

**References:** picoCTF: WPA-ing Out · Tools: Wireshark, aircrack-ng, hashcat, rockyou.txt
