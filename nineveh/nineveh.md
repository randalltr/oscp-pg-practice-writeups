# HTB Nineveh - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-27

---

## 0. Lesson Learned



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

---

## 7. Post-Exploitation

---

## 8. Privilege Escalation

---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix