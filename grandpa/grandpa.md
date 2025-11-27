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