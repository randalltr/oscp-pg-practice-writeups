# HTB Jeeves - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-16

---

## 0. Lesson Learned

If flags and files don't exist when they should, always run `dir /R` to check for hidden data attached to files as NTFS supports **alternate data streams**.

**CrackMapExec:** SMB/AD enum + credential validation + light command exec (legacy, still seen).

**NetExec:** Modern CME replacement; fast SMB/AD enum, cred spraying, tells you where creds work.

**Evil-WinRM:** PowerShell shell over WinRM; works with non-admin creds if allowed, clean and stable.

**PsExec:** SMB service-based exec; requires admin, drops you straight into SYSTEM.

---

## 1. Executive Summary

A penetration test was performed against the target host *Jeeves*. Multiple weaknesses were identified, including exposed administrative services, insecure credential storage, and improper handling of sensitive data. These issues enabled and attacker to gain initial access via a Jenkins service, extract stored credentials, reuse administrator hashes, and escalate privileges to SYSTEM. Full compromise of the host was achieved, including retrieval of user and root proof files.

---

## 2. Scope

- **Target:** 10.129.228.112
- **Environment:** HackTheBox Jeeves (Retired Machine)
- **Testing Window:** 2026-1-15 to 2026-1-16
- **Objective:** Obtain user-level and root-level access

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