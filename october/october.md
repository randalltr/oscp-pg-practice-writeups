# HTB October - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-6-2

---

## 0. Lesson Learned

You must always check manually for sudo permissions and SUID binaries - don't rely on linPEAS for this.

---

## 1. Executive Summary

An external penetration test was conducted against the target system *October*. The assessment resulted in full system compromise, including root-level access, by chaining multiple weaknesses: 

---

## 2. Scope

- **Target:** 10.129.96.113
- **Environment:** HTB October (Retired Machine)
- **Testing Window:** 2026-6-2
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
nmap -p- -sCV 10.129.96.113 -T4 -oA october_initial
```