# Pico Bank — CTF Write-up

| | | | |
|---|---|---|---|
| **Challenge** | Pico Bank | **Category** | Reverse Engineering |
| **Event** | picoMini by CMU-Africa | **Difficulty** | Medium |
| **Author** | Prince Niyonshuti N. | **Platform** | CyLab Academy |
| **Solved** | 2026-11-24 | **Artifacts** | `pico-bank.apk` + web API (`xebec.cylabacademy.net`) |

> **Flag:** `academy{1_l13d_4b0ut_b31ng_s3cur3d_m0b1l3_l0g1n_7f1dd571}`
> Reads: *"I lied about being a secured mobile login"* — the bank's "Security Beyond the Limits" tagline, self-owned.

## 1. TL;DR
An Android banking app hides **half its flag inside the app** (characters disguised as transaction amounts) and serves **the other half from a web API**. Decompile the app to read the hidden data and the login/OTP logic — the credentials and OTP are baked into the client — then send the recovered OTP to the server, which returns the second half.

## 2. The challenge
Pico Bank claims to be "impervious." The flag may live in **more than one place**, a user's credentials leaked "in an unusual way," and we're given a victim profile:
```
Name: Alex Johnson      Email: johnson@picobank.com
DOB:  March 14, 1990    Pet: tricky     Favourite colour: Blue
Last transaction: $345.67
```

## 3. Background
An **APK** is a ZIP; the code compiles to *DEX* bytecode; **jadx** decompiles DEX back to readable **Java**.

> **Client-side checks are not security.** Anything the app must do — check a password, validate an OTP, hide a value — ships inside the APK on the user's device. Anyone can decompile it and read those secrets.

## 4. Recon & decompile
```bash
file pico-bank.apk               # Android package (APK)
jadx -d pico_src pico-bank.apk   # decompile ALL code to .java files
# package: com.example.picobank (ignore android/androidx/kotlin libs)
# classes: Splash, Login, OTP, MainActivity, Transaction(Adapter), Notification
```
A `strings` scan hinted at two things: binary-looking literals (`1100001`, …) and an OTP path (`verifyOtp`, plus a Toast: *"Have you analyzed the server's response when handling OTP requests?"*).

## 5. Flag, part 1 — hidden in the app
In `MainActivity`, each transaction's **amount** is a binary string, and the **order of the list is the flag order**:
```java
new Transaction("Grocery Shopping", "2023-07-21", "$ 1100001", false); // 1100001 = 'a'
new Transaction("Electricity Bill", "2023-07-20", "$ 1100011", false); // 1100011 = 'c'
...  // 27 transactions total
```
Decode each 7-bit binary to ASCII, in list order:
```python
python3 -c "bins='1100001 1100011 1100001 1100100 1100101 1101101 1111001 1111011 110001 1011111 1101100 110001 110011 1100100 1011111 110100 1100010 110000 1110101 1110100 1011111 1100010 110011 110001 1101110 1100111 1011111'.split(); print(''.join(chr(int(b,2)) for b in bins))"
# -> academy{1_l13d_4b0ut_b31ng_
```
> **Why `strings` scrambled this earlier:** `strings` reads the DEX's sorted string *pool*, not the order the code uses the values. The real order only appears in the decompiled transaction list. First rule of `strings`: it finds text, it does not preserve meaning.

## 6. Cracking the login
`Login`'s button handler compares to hardcoded strings:
```java
if ("johnson".equals(username) && "tricky1990".equals(password)) {
    startActivity(new Intent(this, OTP.class));   // success -> OTP screen
}
```
Both map to the profile (the "unusual leak"):
- **Username** `johnson` — the email's local part; the app even greets "Welcome, Johnson."
- **Password** `tricky1990` — pet name **tricky** + birth year **1990**. Plaintext, in the code.

## 7. The OTP and the stubbed server
`OTP.verifyOtp()`:
```java
String endpoint = "your server url/verify-otp";          // dev PLACEHOLDER, never filled in
if (getResources().getString(R.string.otp_value).equals(otp)) { ... }  // OTP is a resource!
... JsonObjectRequest(POST, endpoint, {"otp": otp}) ...
    onResponse: if (response.getBoolean("success"))
                    String flag = response.getString("flag");   // <- flag half #2
```
The OTP value is readable straight out of resources:
```bash
grep otp_value pico_src/resources/res/values/strings.xml
#   <string name="otp_value">9673</string>
```
> **The placeholder URL is the whole point.** `"your server url"` is a stub the dev forgot to replace — and the app was downloaded from `xebec.cylabacademy.net:<port>`. That site *is* the server. The Toast hint is an invitation to send the OTP ourselves and read the JSON.

## 8. Flag, part 2 — from the server
```bash
curl -s -X POST http://xebec.cylabacademy.net:<port>/verify-otp \
  -H "Content-Type: application/json" -d '{"otp":"9673"}'
```
```json
{"success":true,"message":"OTP verified successfully",
 "flag":"s3cur3d_m0b1l3_l0g1n_7f1dd571}",
 "hint":"The other part of the flag is hidden in the app"}
```
The server's own hint confirms the two-location design.

## 9. Assembling the flag
```
part 1 (app transactions): academy{1_l13d_4b0ut_b31ng_
part 2 (server response):                             s3cur3d_m0b1l3_l0g1n_7f1dd571}
------------------------------------------------------------------------------
academy{1_l13d_4b0ut_b31ng_s3cur3d_m0b1l3_l0g1n_7f1dd571}
```

## 10. The chain, tool by tool
| Stage | Tool | What it yielded |
|---|---|---|
| Unpack & triage | unzip, strings | APK is a ZIP; spotted binary blobs + OTP hints |
| Decompile | jadx | Readable Java for Login, OTP, MainActivity |
| Decode part 1 | python (binary→ASCII) | Flag half hidden in transaction amounts, in list order |
| Read login logic | reading decompiled code | Credentials johnson / tricky1990 (pet + birth year) |
| Read OTP + resources | jadx, grep strings.xml | OTP 9673; the stubbed server URL |
| Hit the API | curl | Server returned the flag's second half |

## 11. Takeaways
**Solving:**
- An APK is a ZIP; `jadx` is your decompiler. Read the app's own package; ignore `android/androidx/kotlin` noise.
- `strings` finds text but scrambles order — confirm anything structural by reading the actual code.
- Always check which package a string belongs to: a false `secure` hit here came from okhttp/Java libraries, not the flag.
- A placeholder URL + a "check the server's response" hint means: recover the input and talk to the API yourself with `curl`.

**Security:**
- Never trust the client. Passwords, OTPs, and secret data embedded in a mobile app are readable by anyone who decompiles it.
- Passwords built from personal info (pet name + birth year) are cracked instantly — and this one wasn't even hashed.
- Sensitive values and flags should live server-side, released only after a check the client cannot forge.

**References:** picoMini by CMU-Africa — Pico Bank · Tools: unzip, strings, jadx, grep, python, curl
