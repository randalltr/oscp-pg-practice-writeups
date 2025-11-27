# HTB Grandpa - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-27

**Target:** 10.129.95.233

**Environment:** HTB Grandpa (Retired Machine)

---

## 0. Lesson Learned

You must always keep pivoting, especially when exploits that should work don't. 

---

## 1. Executive Summary

This assessment evaluated the security posture of the target Windows host known as **Grandpa**. During the engagement, multiple critical vulnerabilities were identified, leading to complete system compromise.

Key findings include:

- The target exposed is **IIS 6.0**, an outdated and unsupported web server.
- The server was vulnerable to a known remote code execution flaw (**CVE-2017-7269**) triggered via a malicious *PROPFIND* request.
- Post-exploitation enumeration revealed the **SeImpersonatePrivilege** privilege on the compromised account.
- Privilege escalation was achieved successfully through the **Churrasco** token impersonation technique, granting full **NT AUTHORITY\SYSTEM** control of the host.

At the conclusion of the testing, both **user.txt** and **root.txt** were retrieved, demonstrating full compromise of the system.

---

## 2. Scope

- **Target:** 10.129.95.233
- **Environment:** Isolated penetration testing lab
- **Testing Window:** 2025-11-25 to 2025-11-27
- **Objective:** Obtain user-level and administrative-level access

---

## 3. Methodology

The assessment followed OSCP best-practice methodology:

1. Information Gathering
2. Enumeration
3. Vulnerability Identification
4. Exploitation
5. Post-Exploitation Enumeration
6. Privilege Escalation
7. Capture of Proof Files

---

## 4. Information Gathering

### 4.1 Nmap Scan

A full TCP port scan was conducted:

```
nmap -p- -sCV 10.129.95.233 -T4 -oA grandpa_initial
```

Results:

- Port **80/tcp** open
- Service: **Microsoft IIS 6.0** (Windows Server 2003)

This version is known to contain multiple critical vulnerabilities.

---

## 5. Enumeration

### 5.1 Web Enumeration

Gobuster was used to enumerate directories.

```
gobuster dir -u http://10.129.95.233 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Discovered:

- `/images` - directory listing denied
- `/_private` - 403 Forbidden

No direct browsing access, but findings suggested **WebDAV** behavior.

### 5.2 Technology identification

IIS 6.0 combined with directory behavior indicated potential for **WebDAV exploitation** and known RCE vulnerabilities.

---

## 6. Exploitation

###