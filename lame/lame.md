# HTB Lame - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-19

**Target:** 10.129.24.50

**Environment:** HTB Lame (Retired Machine)

**Assessment Type:** Internal lab penetration test

**Overall Risk:** High

**Outcome:** The tester obtained root access and captured both `user.txt` and `root.txt`.

---

## 1. Executive Summary

This report documents the results of a penetration test performed against **HTB: Lame**, a single-host Linux target. The goal of the assessment was to evaluate the security posture of the system, identify vulnerabilities, and determine whether the tester could obtain full compromise.

The tester identified multiple exposed services, including FTP, SMB, SSH, and DistCC. Several potential exploitation paths were evaluated; however, the successful compromise was achieved via a known **Samba 3.0.20 "usermap script" remote command execution vulnerability**, resulting in immediate **root-level access**.

### 1.1 High-Level Results

- SMB allowed anonymous enumeration and exposed a vulnerable Samba service.
- DistCC on port 3632 was present but not the final exploitation vector.
- vsftpd 2.3.4 backdoor appeared present but unreliable.
- Successful exploitation of Samba 3.0.20 usermap vulnerability -> root shell.

### 1.2 Prioritized Recommendations

1. **Upgrade Samba to a supported version** to remove the usermap script RCE vulnerability.
2. **Disable SMB null sessions** to prevent anonymous enumeration.
3. **Restrict exposed services** (FTP, SMB, DistCC) to untrusted networks.
4. **Implement regular patching** to avoid reliance on legacy, vulnerable service versions.

---

## 2. About the Penetration Test

### 2.1 Objective

The objective of this assessment was to identify vulnerabilities on the Lame target host and determine whether an attacker could gain unauthorized access and escalate privileges to root.

### 2.2 Scope

#### In-scope host:

- 10.129.24.50 (Lame)

#### Out of scope:

- None

#### Testing Window:

- 2025-11-19

### 2.3 Methodology

- Information Gathering
- Service Enumeration
- Vulnerability Identification
- Exploitation
- Privilege Escalation
- Post-Exploitation
- Cleanup

---

## 3. Summary of Technical Findings

| Finding | Severity | Description|
|---------|----------|------------|
| Samba 3.0.20 Usermap Script RCE | Critical | Vulneratble Samba version allows remote command execution as root. |
| SMB Null Session | Medium | Anonymous login and share access were possible. |
| DistCC Remote Execution (not exploited) | Medium | DistCC service exposed, potential RCE, but not used for final compromise. |
| vsftpd 2.3.4 Backdoor (unreliable) | Medium | Backdoor version detected; exploit attempted but not successful. |

---