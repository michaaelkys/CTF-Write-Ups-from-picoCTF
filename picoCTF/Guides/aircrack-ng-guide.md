# aircrack-ng — Quick Guide

A beginner-friendly guide to cracking a captured WPA/WPA2 Wi-Fi handshake, the way it comes up in CTFs (e.g. *WPA-ing Out*). Focused on the **offline** workflow: you're given a `.pcap` and a wordlist, and you recover the password.

> **Only run this against networks you own or are authorised to test.** Capturing or cracking other people's Wi-Fi is illegal in most places.

## What aircrack-ng does
`aircrack-ng` takes a captured **WPA 4-way handshake** and tries every password in a wordlist against it. For each candidate it derives a key and checks whether it matches the handshake. When one matches, that's the Wi-Fi password.

It does **not** send anything to the network. All the work is done offline on your machine, so there's nothing to detect and no time limit.

## The WPA 4-way handshake
When a device connects to WPA/WPA2 Wi-Fi, it exchanges **4 EAPOL messages** with the router to prove both know the password, without sending it. If you've captured all 4, you have everything needed to test guesses offline.

You need **all four messages**. A partial handshake can't be cracked.

## Step 1 — Confirm the handshake is in the capture
Open the file in Wireshark and apply this display filter:
```
eapol
```
You should see 4 packets (Message 1 of 4 → 4 of 4). While you're there, the network name (SSID) is in the beacon frames:
```
wlan.fc.type_subtype == 0x08
```
Or check from the terminal, aircrack lists any networks and whether a handshake was found:
```bash
aircrack-ng capture.pcap
```
Look for `(1 handshake)` next to the network.

## Step 2 — Get a wordlist
The standard CTF wordlist is **`rockyou.txt`** (~14 million real leaked passwords). On Kali it's here, often gzipped:
```bash
ls /usr/share/wordlists/
sudo gunzip /usr/share/wordlists/rockyou.txt.gz    # decompress if it ends in .gz
```
aircrack-ng cannot read a `.gz` directly, so decompress it first.

## Step 3 — Crack
```bash
aircrack-ng capture.pcap -w /usr/share/wordlists/rockyou.txt
```
| Part | Meaning |
|---|---|
| `capture.pcap` | the capture file holding the handshake |
| `-w <file>` | the wordlist to try |

If the capture has more than one network, aircrack lists them and asks which to attack, pick the number.

## Step 4 — Read the result
```
                          KEY FOUND! [ mickeymouse ]
```
The text in brackets is the password. In a CTF, that's usually the flag (wrap it in the required format, e.g. `academy{mickeymouse}`).

## Command reference
```bash
# List networks + whether a handshake was captured
aircrack-ng capture.pcap

# Crack with a wordlist
aircrack-ng capture.pcap -w rockyou.txt

# Target one network by name or BSSID (if several are present)
aircrack-ng capture.pcap -w rockyou.txt -e "NetworkName"
aircrack-ng capture.pcap -w rockyou.txt -b AA:BB:CC:DD:EE:FF
```

## hashcat alternative (faster, uses your GPU)
```bash
# 1. Convert the handshake to hashcat's format (hcxtools)
hcxpcapngtool -o hash.hc22000 capture.pcap

# 2. Crack (mode 22000 = WPA)
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt

# 3. Show the cracked password afterwards
hashcat -m 22000 hash.hc22000 --show
```

## Troubleshooting
| Symptom | Cause / fix |
|---|---|
| `No valid WPA handshakes found` | The capture is missing EAPOL messages. Check with the `eapol` filter; you need all 4 |
| `fopen(dictionary) failed` | Wrong wordlist path, or it's still `.gz`. Decompress and give the full path |
| Runs through the whole list, no key | The password isn't in this wordlist. Try a bigger one, or rules (`hashcat -r`) |
| Very slow | Use hashcat with a GPU, or a smaller/targeted wordlist |

## Why this works (and how to defend)
- The handshake is broadcast over the air, so anyone nearby can capture it. Cracking then happens offline, invisibly.
- A weak or reused password (anything in `rockyou`) falls in seconds.
- **Defences:** a long random passphrase, **WPA3** (its SAE handshake resists offline dictionary attacks), or enterprise 802.1X auth instead of a shared password.

---
*For authorised lab / CTF use only.*
