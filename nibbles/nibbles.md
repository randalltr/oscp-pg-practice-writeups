# HTB Nibbles - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-1

**Target:** 10.129.20.119

**Environment:** HTB Nibbles (Retired Machine)

---

## 0. Lesson Learned

Sometimes the answer is easier than you are making it - don't try to catch a privileged reverse shell when you can just upgrade the current shell to root.

---

## 1. Executive Summary

A security assessment was performed against hte target host **10.129.20.119**. During the engagement, multiple weaknesses were identified that allowed full compromise of the system, including user-level access and eventual root privilege escalation. The primary attack vector was a vulnerable instance of **Nibbleblog 4.0.3** susceptible to an **Arbitrary File Upload (CVE-2015-6967)**. Privilege escalation was achieved by abusing a misconfigured **sudo-executable script (monitor.sh)**.

The assessment resulted in full system compromise.

---

## 2. Scope

- **Target:** 10.129.20.119
- **Environment:** HackTheBox (legal/authorized)
- **Testing Window:** 2025-11-28 to 2025-12-1
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