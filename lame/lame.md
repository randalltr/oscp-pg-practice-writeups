# HTB Lame - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-20

**Target:** 10.129.24.50

**Environment:** HTB Lame (Retired Machine)

---

## 1. Executive Summary

This report documents the results of a penetration test performed against **HTB: Lame**, a single-host Linux target. The goal of the assessment was to evaluate the security posture of the system, identify vulnerabilities, and determine whether the tester could obtain full compromise.

The tester identified multiple exposed services, including FTP, SMB, SSH, and DistCC.  The tester successfully obtained an initial low-privileged shell via a DistCC remote command execution vulnerability, gaining access as the `daemon` user and capturing the user flag. However, privilege escalation was not achievable from this foothold. The tester then identified a known **Samba 3.0.20 "usermap script" remote command execution vulnerability**, resulting in immediate **root-level access**.

### 1.1 High-Level Results

- Initial foothold obtained via DistCC RCE -> shell as `daemon` and user flag captured.
- Privilege escalation attempts from user `daemon` were unsuccessful.
- vsftpd 2.3.4 backdoor appeared present but unreliable.
- SMB allowed anonymous enumeration and exposed a vulnerable Samba service.
- Successful exploitation of Samba 3.0.20 usermap vulnerability -> root shell and root flag captured.

### 1.2 Prioritized Recommendations

1. **Upgrade Samba to a supported version** to remove the usermap script RCE vulnerability.
2. **Disable SMB null sessions** to prevent anonymous enumeration.
3. **Restrict exposed services** (FTP, SMB, DistCC) to untrusted networks.
4. **Implement regular patching** to avoid reliance on legacy, vulnerable service versions.

---

## 2 Objective

The objective of this assessment was to identify vulnerabilities on the HTB Lame target host and determine whether an attacker could gain unauthorized access and escalate privileges to root.

## 3. Summary of Technical Findings

| Finding | Severity | Description|
|---------|----------|------------|
| Samba 3.0.20 Usermap Script RCE | Critical | Vulnerable Samba version allows remote command execution as root. |
| DistCC Remote Execution (successful foothold) | High | DistCC service exposed, allows execution of arbitrary command and provides shell as `daemon`. |
| SMB Null Session | Medium | Anonymous login and share access were possible. |
| vsftpd 2.3.4 Backdoor (unreliable) | Medium | Backdoor version detected; exploit attempted but not successful. |

---

## 4. Attack chain:

1. Enumerated DistCC (port 3632) and confirmed RCE.
2. Executed DistCC exploit -> obtained shell as user `daemon`.
3. Captured **user.txt**.
4. Attempted privesc paths (socat, SUID search, at-job abuse) but none succeeded.
5. Pivoted to SMB enumeration and identified a vulnerable Samba version.
6. Executed Samba usermap script exploit -> obtained **root shell**.
7. Captured **root.txt**.

### 4.1 Flags captured:

 - user.txt (from `daemon` DistCC shell)
 - root.txt (from Samba RCE)

 ---

 ## 5. Methodology

 ### 5.1 Information Gathering

 A full TCP port scan was launched:

```
nmap -p- -sCV 10.129.24.50 -T4 -oA lame_initial
```

Key findings:

- FTP (21)
- SSH (22)
- SMB (139/445)
- DistCC (3632)

### 5.2 Service Enumeration