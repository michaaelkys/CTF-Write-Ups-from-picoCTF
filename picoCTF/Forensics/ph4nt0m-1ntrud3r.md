# Ph4nt0m 1ntrud3r — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Ph4nt0m 1ntrud3r | **Category** | Forensics |
| **Event** | picoCTF 2025 | **Difficulty** | Easy |
| **Author** | Prince Niyonshuti N. | **Platform** | CyLab Academy |
| **Solved** | 2026-11-28 | **File** | `myNetworkTraffic.pcap` (22 TCP packets) |

> **Flag:** `academy{1t_w4snt_th4t_34sy_tbh_4r_a8139a41}`
> *"it wasn't that easy tbh"*. The pieces were all there, just scrambled in time.

## 1. TL;DR
The PCAP holds **22 TCP packets**. Some of them carry a small payload, and each payload is a **Base64 fragment** of the flag. The packets are **out of order**: their list position doesn't match their timestamps. Filtering to the packets that carry data, sorting them **by time**, and decoding each fragment rebuilds the flag.

## 2. The challenge
> *"...The attacker has cleverly concealed his moves in **well timely manner**. Dive into the network traffic, **apply the right filters**..."*

Two direct hints:
- **"timely manner"** → the **order** depends on timestamps.
- **"apply the right filters"** → only some packets matter.

## 3. Analysis

### 3.1 First look
```bash
wireshark myNetworkTraffic.pcap
```
22 packets, all **TCP**. Most have no data. The interesting ones carry a **12-byte payload**.

### 3.2 Filter to the data
```
tcp.len > 0
```
`tcp.len` is the size of the TCP payload. Filtering out the empty packets leaves only the ones carrying fragments.

### 3.3 Read the payloads
Right-click → **Follow → TCP Stream** (or look at each packet's bytes pane). Each payload is short text like:
```
bnRfdGg0dA==
```
| Clue | Meaning |
|---|---|
| `A–Z a–z 0–9`, ends `=` / `==` | **Base64** (not hex: hex only uses `0–9A–F`) |
| 12 characters | Base64 of **7 bytes**, so each fragment is a 7-character piece of the flag |

### 3.4 One fragment "wouldn't decode"
```
3bnRfdGg0dA==    ← 13 characters, fails
 bnRfdGg0dA==    ← 12 characters, decodes to "nt_th4t"
```
Valid Base64 is always a **multiple of 4** characters. 13 isn't, so one character was extra. A stray `3` from the next byte in the Follow-Stream view got copied along with the string.

### 3.5 Decoded fragments (read in packet-number order: scrambled)
```
_34sy_t   bh_4r_a   academy   8139a41   {1t_w4s   ...
```
The pieces are all there, but in packet-number order they make no sense.

### 3.6 Re-order by time
Sorting by the **Time** column (instead of **No.**) puts the fragments in the order the flag was written:
```
academy  {1t_w4s  nt_th4t  _34sy_t  bh_4r_a  8139a41  }
```
```
academy{1t_w4snt_th4t_34sy_tbh_4r_a8139a41}
```

## 4. The key decision: pick the right tool
| Approach | Why it fails / wins |
|---|---|
| `strings myNetworkTraffic.pcap \| grep academy` | The flag is Base64-encoded and split across packets, so it never appears whole |
| Follow TCP Stream and decode the whole stream at once | Fragments are concatenated in **capture order**, which is scrambled |
| Read packets in **No.** order | Gives every fragment, but in the wrong sequence |
| Filter `tcp.len == 12` only | Gets most fragments but can miss a shorter one, like the closing `}` |
| **Filter `tcp.len > 0` → sort by Time → decode each piece (chosen)** | Every fragment, in the right order |

> **The core insight:** in network forensics, **packet number ≠ event order**. The capture lists packets in the order they were recorded, but the timestamps record when they were actually sent. When the challenge stresses timing, sort by time.

## 5. The solver
**Wireshark (what we used):**
```
Display filter:  tcp.len > 0
Sort by:         Time column
Decode:          each payload with CyberChef → From Base64
```
**Command-line equivalent:**
```bash
tshark -r myNetworkTraffic.pcap -Y "tcp.len > 0" \
       -T fields -e frame.time_epoch -e tcp.payload | sort -n
# column 2 is the payload in hex. Hex → text gives each Base64 fragment, then decode in this order
```
> **Gotcha:** when copying Base64 out of Wireshark's stream view, check the length. Base64 is always a multiple of 4 characters. Our "undecodable" fragment was 13 characters because a neighbouring byte got copied too. Removing it fixed the decode.

## 6. The run
```
academy | {1t_w4s | nt_th4t | _34sy_t | bh_4r_a | 8139a41 | }
→ academy{1t_w4snt_th4t_34sy_tbh_4r_a8139a41}
```

## 7. Takeaways
**Solving:**
- **Sort by time when the challenge mentions timing.** Capture order can be deliberately shuffled.
- **Filter to the data first:** `tcp.len > 0` (or `udp.length > 8`) removes empty handshake and ACK packets.
- **Don't over-tighten filters.** `tcp.len == 12` would miss a shorter final fragment.
- **Check Base64 length (multiple of 4)** before deciding a fragment is "broken". Usually it's a copy error.
- **Carried forward:**
  - *Riddle Registry / RED*: spot Base64 by its alphabet and `=` padding (and don't confuse it with hex).
  - *DISKO 1*: try the cheap check first. `strings` fails here, which tells you the data is split or encoded.

**Security (SOC angle):**
- Splitting stolen data into **small encoded chunks** is a real exfiltration technique. Each packet looks harmless, and no single one contains the secret.
- Detection ideas: many small, uniform-size payloads to one host, payloads that are **pure Base64**, and timestamps that don't match the packet order (a sign of a crafted or reordered capture).
- Always analyse events on a **timeline**. Investigators rebuild what happened from timestamps, not file order.

## 8. Pivot Log
| # | Pivot | Trigger |
|---|---|---|
| 1 | Filter to packets **with data** | 22 TCP packets, most empty; "apply the right filters" |
| 2 | Decode payloads as **Base64** | Base64 alphabet, `=` padding, 12-character fragments |
| 3 | Fix the "undecodable" fragment | 13 characters isn't a multiple of 4, so drop the stray `3` |
| 4 | **Sort by Time**, not packet number | Fragments made no sense in capture order; "well timely manner" |
| 5 | Include the short **closing fragment** | The flag needs a `}`, which `tcp.len == 12` would filter out |

**References:** picoCTF 2025: Ph4nt0m 1ntrud3r · Tools: Wireshark, tshark, CyberChef (From Base64)
