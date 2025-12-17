# HTB Devel - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-17

---

## 0. Lesson Learned

Classic ASP (`.asp`) is enabled out of the box on **IIS 5 / 6**, and is optional but often not installed on **IIS 7+**. On newer IIS, `.aspx` is the safer bet.

---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 10.129.11.44
- **Environment:** HackTheBox Devel (Retired Machine)
- **Testing Window:** 2025-12-17
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