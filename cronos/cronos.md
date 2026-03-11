# HTB Cronos - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-11

---

## 0. Lesson Learned



---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 10.129.227.211
- **Environment:** HTB Cronos (Retired Machine)
- **Testing Window:** 2026-3-11
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

A full TCP-scan was conducted with Nmap:

```
nmap -p- -sCV 10.129.227.211 -T4 -oA cronos_tcp
```