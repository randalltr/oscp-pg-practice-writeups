# HTB Valentine - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-14

---

## 0. Lesson Learned

Identifying the age of outdated Linux distributions helps find relevant vulnerabilities.

When privesc stalls, run linpeas and look for the red text with yellow background.

---

## 1. Executive Summary

A penetration test was conducted against the target host *Valentine*. The assessment identified multiple critical weaknesses related to outdated services and cryptographic misconfigurations. Exploitation of the **OpenSSL Heartbleed vulnerability (CVE-2014-0160)** resulted in credential disclosure, which enabled SSH access to the system. Further post-exploitation enumeration revealed a misconfigured `tmux` socket running as root, ultimately leading to full system compromise.

---

## 2. Scope

- **Target:** 10.129.232.136
- **Environment:** HackTheBox Valentine (Retired Machine)
- **Testing Window:** 2026-1-12 to 2026-1-14
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

An initial full TCP port scan was conducted using Nmap:

```
nmap -p- -sCV 10.129.232.136 -T4 -oA valentine_initial
```

The following services were identified:

- SSH on port 22
- HTTP on port 80
- HTTPS on port 443

---

## 5. Enumeration

### SSH (Port 22)

- OpenSSH 5.9p1
- Legacy DSA host key observed

### Web Services (Ports 80/443)

- Apache httpd 2.2.22 running on Ubuntu
- HTTPS enabled with an expired SSL certificate

Directory enumeration using Gobuster revealed several directories of interest:

- `/decode`
- `/encode`
- `/dev`

The `/decode` and `/encode` endpoints appeared to perform Base64 operations but did not execute server-side input.

---

## 6. Initial Access

Given the age of the Apache and OpenSSL stack, the HTTPS service was tested for the Heartbleed vulnerability using the following Nmap script:

```
nmap -sV -p 443 --script=ssl-heartbleed.nse 10.129.232.136
```

The service was confirmed vulnerable.

A public Heartbleed proof-of-concept was executed, resulting in disclosure of Base64-encoded data. Decoding the leaked data revealed the string:

```
heartbleedbelievethehype
```

The value was later confirmed to be a valid SSH passphrase.

---

## 7. Post-Exploitation

---

## 8. Privilege Escalation

---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix