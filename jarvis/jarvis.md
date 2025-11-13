# HTB Jarvis - OSCP-Style Penetration Test Report
**Author:** RandallTr
**Target:** 10.129.229.137
**Environment:** HTB (Retired Machine)
**Date:** 2025-11-13

---

## 1. High-Level Summary

A penetration test was conducted against the *Jarvis* host on the HackTheBox network. Multiple critical vulnerabilities were discovererd, including:
- SQL Injection in the `/room.php?cod=` parameter.
- Plaintest database credentials stored on the web server.
- Weak database configuration allowing SQL file reads via `LOAD_FILE()`.
- Outdated phpMyAdmin version vulnerable to authenticated LFI -> RCE.
- Privilege escalation through misconfigured `sudo` fules and a SUID `systemctl` binary.

Successful exploitation yielded:
- Remote code execution as `www-data`.
- Privilege escalation to the user `pepper`.
- Full root compromise of hte system.

All flags were retrieved.

---

## 2. Executive Recommendations

**1. Patch phpMyAdmin** to a current supported version and disable risky features such as direct SQL execution for untrusted users.
**2. Sanitize SQL input** or use prepared statements throughout the application.
**3. Remove plaintext credentials from webroot**, implement proper secrets management.
**4. Disable LOAD_FILE** and secure MySQL file privileges.
**5. Enforce strict sudo restrictions** and remove dangerous SUID binaries.
**6. Implement web application firewalls (WAF)** to mitigate SQLi and LFI attempts.

---

## 3. Methodology

A standard OSCP-aligned methodology was followed:

### 3.1 Information Gathering
- Network discovery
- Service enumeration
- Virtual host enumeration
- Directory brute forcing

### 3.2 Vulnerability Identification
- SQL injection testing
- phpMyAdmin exploit research
- MySQL privilege abuse
- SUID binary and sudo enumeration

### 3.3 Exploitation
- SQL injection to dump credentials
- phpMyAdmin authenticated LFI -> RCE
- Reverse shell deployment

### 3.4 Privilege Escalation
- Lateral movement to `pepper` via vulnerable `simpler.py`
- Full root via custom SUID `systemctl` service enumeration

---