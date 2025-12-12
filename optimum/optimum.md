# HTB Optimum - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-12

**Target:** 10.129.13.67

**Environment:** HTB Optimum (Retired Machine)

---

## 0. Lesson Learned

Each Windows version has a small cluster of kernel vulnerabilities associated with it. You will likely need to search Google many times to find them all (or check the appendix).

---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 10.129.13.67
- **Environment:** HackTheBox (legal/authorized)
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

## 4. Information Gathering

---

## 5. Enumeration

---

## Appendix

### Windows Kernel Exploit Clusters

#### Windows 7 Server 2008 R2 (NT 6.1)

- **MS11-046** - afd.sys
- **MS13-053** - win32k.sys
- **MS13-081** - win32k.sys
- **MS14-058** - win32k.sys (shared with 2012 R2)

Less common:

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

#### Windows Server 2003 / XP (NT 5.x)

- **MS08-067**
- **MS09-050**
- **MS10-015**
- **MS11-080**

#### Windows Server 2019 / Windows 10 Modern Builds

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