# HTB Arctic - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-12

---

## 0. Lesson Learned

Rely on your own notes - especially if you have seen something like this before.

---

## 1. Executive Summary

A penetration test was conducted against the target host *Arctic* to assess its security posture. The assessment identified a vulnerable and outdated Adobe ColdFusion service that allowed remote command execution, leading to an initial foothold. Further misconfigurations enabled privilege escalation to SYSTEM-level access.

Complete compromise of the system was achieved, demonstrating critical risk to the target environment.

---

## 2. Scope

- **Target:** 10.129.1.125
- **Environment:** HackTheBox Arctic (Retired Machine)
- **Testing Window:** 2026-1-9
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

A full TCP port scan was performed to identify exposed services:

```
nmap -p- -sCV 10.129.1.125 -T4 -oA arctic_initial
```

**Results:**
- 135/tcp - Microsoft Windows RPC
- 8500/tcp - HTTP (JRun Web Server)
- 49154/tcp - Microsoft Windows RPC

Port 8500 was selected for further investigation.

---

## 5. Enumeration

The HTTP service on port 8500 was manually browsed and directory listing was found to be enabled with observed directories:

- `/CFIDE`
- `/cfdocs`

Accessing the following URL revealed a ColdFusion Administrator portal:

```
http://10.129.1.125:8500/CFIDE/Administrator
```

The application version was identified as **Adobe ColdFusion 8**, a version known to contain multiple critical vulnerabilities.

Exploit research was conducted using Exploit Database, identifying a Remote Command Execution vulnerability applicable to this version.

---

## 6. Initial Access

A ColdFusion 8 Remote Command Execution vulnerability (CVE-2009-2265) was exploited using public exploit **EDB-50057**.

---

## 7. Post-Exploitation

---

## 8. Privilege Escalation

---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix

ColdFusion 8 Remote Command Execution vulnerability (CVE-2009-2265):
[https://www.exploit-db.com/exploits/50057](https://www.exploit-db.com/exploits/50057)