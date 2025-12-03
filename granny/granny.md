# HTB Granny - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-3

**Target:** 10.129.95.234

**Environment:** HTB Granny (Retired Machine)

---

## 0. Lesson Learned

Sometimes you need to look on page 2 of Google search results to find the PoC.

---

## 1. Executive Summary

A penetration test was performed against the HTB Granny target system. The assessment successfully achieved full system compromise, including privilege escalation to **NT AUTHORITY\SYSTEM**. The primary attack vectors were:

- **Remote Code Execution** via **IIS 6.0 WebDAV PROPFIND Buffer Overflow** (CVE-2017-7269)
- **Privilege Escalation** via **Churrasco (Token Kidnapping)** exploiting **SeImpersonatePrivilege**

User and system-level flags were obtained, demonstrating full compromise of the host.

---

## 2. Scope

**Target:** 10.129.95.234
**Allowed Techniques:** Full exploitation permitted
**Prohibited Actions:** None (lab environment)

---

## 3. Methodology

Testing followed standard OSCP methodology:

1. **Enumeration:** Port scanning, service identification, web content review
2. **Vulnerability Identification:** Matching service versions to known exploits
3. **Exploitation:** Gaining initial access
4. **Privilege Escalation:** Evaluating local privileges and token abuse
5. **Post-Exploitation:** Flag acquisition, system confirmation

---

## 4. Initial Enumeration

### Nmap Scan

A full TCP scan was performed:

```
nmap -p- -sCV 10.129.95.234 -T4 -oA granny_initial
```

Revealing the following:

- **Port 80/tcp - Microsoft IIS 6.0**
- "Under Construction" page displayed

### Gobuster Scan

A directory scan was performed:

```
gobuster dir -u http:/10.129.95.234 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Revealing:

- `/images/` directory was identified

---

## 5. Vulnerability Identification

IIS 6.0 WebDAV on Windows Server 2003 is known to be vulnerable to the **PROPFIND buffer overflow (CVE-2017-7269)**.

Based on version banners and OS detection, the following critical vulnerability applied:

**CVE-2017-7269 - IIS 6.0 WebDAV Buffer Overflow RCE**

Triggering a crafted **PROPFIND** request leads to remote code execution. A publicly-available PoC was used to test exploitability.

---

## 6. 