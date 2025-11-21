# HTB Blue - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-21

**Target:** 10.129.23.90

**Environment:** HTB Blue (Retired Machine)

---

## 1. Executive Summary

A penetration test was conducted against the target host at 10.129.23.90. The objective was to identify exploitable vulnerabilities, gain unauthorized access, and obtain proof-of-access artifacts (user and administrator flags).

During testing, open SMB services (TCP 445) combined with the OS fingerprint (**Windows 7 SP1**) indicated a high likelihood of exposure to the **MS17-010 EternalBlue** vulnerability. After confirming the vulnerability through SMB scanning, remote code execution was achieved and full SYSTEM-level access was obtained.

---

## 2. Scope

- **In-scope host:** 10.129.23.90
- **Assessment format:** Black-box
- **Allowed tools:** All standard OSCP-approved tooling
- **Objective:** Gain shell access, escalate privileges, and retrieve flags.