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