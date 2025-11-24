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

---

### 4.2 Web Application Enumeration

Directory fuzzing discovered a `/dev/` directory:

```
gobuster dir -u http://10.129.22.73 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html
```

Inside `/dev/`, a **phpbash** instance was available:

- `/dev/phpbash.php`

This provided an **interactive web shell** as the `www-data` user.

**Impact:** Direct code execution on the host.
