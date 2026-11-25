# Chronohack — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Chronohack | **Category** | Reverse Engineering |
| **Event** | picoCTF 2025 | **Difficulty** | Medium |
| **Author** | Theoneste Byagutangaza | **Platform** | CyLab Academy |
| **Solved** | 2026-11-23 | **Target** | `nc chatelaine.cylabacademy.net <port>` |

> **Flag:** `academy{UseSecure#$_Random@j3n3r@T0rs2627fc3a}`
> The flag itself is the lesson: *"Use Secure Random Generators."*

## 1. TL;DR
The server builds a secret 20-character token by seeding Python's random number generator with the **current clock time in milliseconds**. Because we can know roughly what time it is on the server, we can rewind the generator to the same starting point, regenerate the identical token on our own machine, and type it in. The whole challenge is turning *"roughly what time it is"* into *"the exact millisecond,"* solved by syncing our clock and sweeping a small window of candidate times.

## 2. The challenge
We get a network service that greets us and lets us guess a token. Correct token → flag. We get **50 guesses** per connection.

```
$ nc chatelaine.cylabacademy.net <port>
Welcome to the token generation challenge!
Can you guess the token?

Enter your guess for the token (or exit):
```

Brute force is hopeless: 62²⁰ possibilities. The weakness is in *how* the token is made.

## 3. Background: PRNGs and seeds
A **PRNG** (pseudo-random number generator) computes a stream of values that only *look* random. It starts from a **seed**.

> **Key fact:** the same seed always produces the exact same sequence. "Random" only means "hard to predict *if you don't know the seed.*" If you know the seed, there's no randomness left.

So the security depends entirely on the seed being unpredictable. This is the **insecure randomness** vulnerability.

## 4. The leaked source
```python
def get_random(length):
    alphabet = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    random.seed(int(time.time() * 1000))   # <-- the vulnerability
    s = ""
    for i in range(length):
        s += random.choice(alphabet)
    return s
```
- `time.time()` = seconds since 1970 (Unix epoch). `× 1000` → **milliseconds**. So the seed is just "the current time in ms."
- 20 × `random.choice` = the token, fully determined by that seed.

Winning loop: we get 50 guesses; submit the correct token within them.

## 5. The vulnerability
The token is seeded with the current time — **not a secret**. Our own clock gives (nearly) the same value. Reconstruct the seed → reconstruct the token.

Attack in 3 steps:
1. **Connect** and note the time in epoch ms.
2. **Reproduce** the token with the same `seed → 20×random.choice` routine.
3. **Submit** it. If our guessed time matches the server's seed, the tokens are identical.

## 6. The real problem: hitting the exact millisecond
The seed is in **milliseconds** → 1000 different seeds per second. And we only get **50 guesses per connection** (a 50 ms window).

**Strategy — tight window + reconnect sweep:**
- Each connection, estimate server seed time `t`, try seeds `t-25 … t+24` (50 guesses).
- Miss → reconnect: a new token, a new seed, and a **fresh 50 guesses**.
- Shift the window center outward each reconnect (0, −50, +50, −100, …) to hunt the (roughly constant) clock offset.

## 7. The pivots
| # | Symptom | Diagnosis | Pivot |
|---|---|---|---|
| 1 | No hit even at ±200 ms | VirtualBox VM clock drift | Resync the VM clock to real time |
| 2 | Still missing after rough sync | Timestamp measured at connect baked in the server's Python start-up delay (jittery) | Anchor the timestamp right **after** the welcome text (printed just before the seed) |
| 3 | Coarse HTTP `Date`-header sync only accurate to 1 second | 1 s precision is too loose for a ms seed | Sync to millisecond precision via **NTP**, in-script, no admin rights |

> **Why NTP won:** it corrects our clock to real time within milliseconds. Once our time matched real time, and real time matched the server's clock, the first few windows landed on the seed.

## 8. The final exploit
```python
#!/usr/bin/env python3
import random, time, socket, struct
from pwn import remote, context
context.log_level = "warn"

HOST = "chatelaine.cylabacademy.net"
PORT = 0  # set per instance
ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
LEN  = 20
HALF = 25                                   # 50 guesses => [center-25, center+24]

def ntp_offset_ms(host="pool.ntp.org"):
    """How far our clock is from real time, in ms (without changing the system clock)."""
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(5)
    s.sendto(b"\x1b" + 47 * b"\0", (host, 123))
    local_before = time.time()
    data, _ = s.recvfrom(1024); s.close()
    local_after = time.time()
    secs, frac = struct.unpack("!12I", data)[10:12]
    ntp_ms = (secs - 2208988800 + frac / 2**32) * 1000
    local_mid_ms = (local_before + local_after) / 2 * 1000
    return ntp_ms - local_mid_ms

try:
    OFFSET = ntp_offset_ms()
    print(f"[*] NTP clock offset = {OFFSET:+.0f} ms (applied in-script)")
except Exception as e:
    OFFSET = 0.0
    print(f"[!] NTP failed ({e}); using system clock")

def gen(seed):
    random.seed(seed)
    return "".join(random.choice(ALPHABET) for _ in range(LEN))

def sweep():
    yield 0
    step = 50
    while True:
        yield -step; yield step
        step += 50

def attempt(center_off):
    io = remote(HOST, PORT)
    io.recvuntil(b"guess the token?")                 # anchor just after welcome
    t = int(time.time() * 1000 + OFFSET) + center_off
    for d in range(-HALF, HALF):
        io.sendlineafter(b"exit):", gen(t + d).encode())
        if b"Congratulations" in io.recvline():
            print("\n[+] HIT at window t%+d ms" % center_off)
            print(io.recvall(timeout=2).decode())
            io.close(); return True
    io.close(); return False

tries = 0
for off in sweep():
    print(f"[*] window center = t{off:+d} ms   (reconnect #{tries})")
    tries += 1
    if attempt(off):
        break
```

## 9. Capturing the flag
```
[*] NTP clock offset = +261 ms (applied in-script)
[*] window center = t+0 ms   (reconnect #0)
 ...
[*] window center = t-150 ms  (reconnect #5)

[+] HIT at window t-150 ms
academy{UseSecure#$_Random@j3n3r@T0rs2627fc3a}
```
The `t-150 ms` hit means the server's seed sat ~150 ms behind our NTP-corrected estimate — the steady offset from network travel time.

## 10. Takeaways
**Solving:**
- "Seeded with the time" = predictable output.
- When timing matters, your clock is part of the exploit — sync it (NTP), measure as close to the event as possible.
- A hard attempt-limit doesn't block a search; reconnecting resets the budget.

**Security:**
- Never seed a PRNG with time (or anything guessable) for tokens, passwords, session IDs, or keys.
- Use a cryptographically secure source: Python's `secrets` module (`secrets.token_hex()`) or `os.urandom()`.

```python
# Vulnerable:
random.seed(int(time.time() * 1000))
token = "".join(random.choice(alphabet) for _ in range(20))

# Secure:
import secrets
token = "".join(secrets.choice(alphabet) for _ in range(20))
```

**References:** picoCTF 2025 — Chronohack · epochconverter.com · Snyk Learn: Insecure Randomness
