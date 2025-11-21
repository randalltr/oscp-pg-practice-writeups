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

---

## 3. Methodology

This methodology follows the OSCP exam structure:

1. Reconnaissance
2. Enumeration
3. Vulnerability Analysis
4. Exploitation
5. Privilege Escalation
6. Proof of Access

All steps are fully documented below.

---

## 4. Reconnaissance

### 4.1 Full Port Scan

A full TCP scan detected several Microsoft RPC and SMB services.

```
nmap -p- -sCV 10.129.23.90 -T4 -oA blue_initial
```

The relevant finding was:

- 445/tcp - microsoft-ds - Windows 7 Professional SP1

The combination of:

- Windows 7 SP1
- SMB service
- No message signing requirement

immediately suggests investigating **MS17-010**.

---

## 5. Enumeration

### 5.1 SMB Vulnerability Scan

```
nmap --script smb-vuln* -p445 10.129.23.90
```
The scan confirmed the presence of **MS17-010 EternalBlue**.

Multiple Python proof-of-concept exploits were available, but due to deprecated Python2 dependencies and byte-handling errors, manual execution was not feasible.

---

## 6. Exploitation