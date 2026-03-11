# HTB Resolute - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-11

---

## 0. Lesson Learned



---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 10.129.96.155
- **Domain:** megabank.local
- **Environment:** HackTheBox Resolute (Retired Machine)
- **Testing Window:** 2026-3-10
- **Objective:** Complete domain compromise

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

Full TCP Port Scan:

```
nmap -p- -sCV 10.129.96.155 -T4 -oA resolute_tcp
```

Key services identified: