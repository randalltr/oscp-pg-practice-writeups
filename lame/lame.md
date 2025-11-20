# HTB Lame - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-19

**Target:** 10.129.24.50

**Environment:** HTB Lame (Retired Machine)

**Assessment Type:** Internal lab penetration test

**Overall Risk:** High

**Outcome:** The tester obtained root access and captured both `user.txt` and `root.txt`.

## 1. Executive Summary

THis report documents the results of a penetration test performed against **HTB: Lame**, a single-host Linux target. The goal of the assessment was to evaluate the security posture of the system, identify vulnerabilities, and determine whether the tester could obtain full compromise.

The tester identified multiple exposed services, including FTP, SMB, SSH, and DistCC. Several potential exploitation paths were evaluated; however, the successful compromise was achieved via a known **Samba 3.0.20 "usermap script" remote command execution vulnerability**, resulting in immediate **root-level access**.

---

