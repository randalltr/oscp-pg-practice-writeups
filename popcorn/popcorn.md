# HTB Popcorn - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-8

**Target:** 10.129.14.108

**Environment:** HTB Popcorn (Retired Machine)

---

## 0. Lesson Learned

Look deeper. When you couldn't manipulate the torrent file, the answer lied in editing the png screenshot of the torrent.

---

## 1. Executive Summary

The objective of this engagement was to perform a controlled penetration test against the **HTB Popcorn** host. The goal was to identify exploitable vulnerabilities, achieve user and root compromise, and document all findings with reproducible steps.

The assessment was successful. A chain of web application vulnerabilities in the *Torrent Hoster* software allowed authenticated file upload manipulation, leading to remote code execution (RCE). Privilege escalation was achieved via a vulnerable 2.6.31 Linux kernel using the **full-nelson** exploit. Full system compromise was obtained.

---

---

## 2. Scope

- **Target:** 10.129.14.108
- **Environment:** HackTheBox (legal/authorized)
- **Testing Window:** 2025-12-4 to 2025-12-8
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

An initial port scan revealed two open services: SSH and Apache 2.2.12.

### Nmap Scan

```
nmap -p- -sCV 10.129.14.108 -T4 -oA popcorn_initial
```

*Result:*

- 22/tcp OpenSSH 5.1p1
- 80/tcp Apache HTTP 2.2.12

Apache default page showed no useful content. Virtual host **popcorn.htb** added to `/etc/hosts/`.

---