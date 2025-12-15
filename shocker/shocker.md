# HTB Shocker - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-15

---

## 0. Lesson Learned

Dirbuster's directory-2.3-medium.txt list is excessively long and may return false negatives. Use dirb/common.txt for initial web enumeration instead.

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

