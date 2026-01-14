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

Given the age of the Apache and OpenSSL stack, the HTTPS service was tested for the Heartbleed vulnerability using the following Nmap script:

```
nmap -sV -p 443 --script=ssl-heartbleed.nse 10.129.232.136
```

The service was confirmed vulnerable.

A public Heartbleed proof-of-concept was executed, resulting in disclosure of Base64-encoded data. Decoding the leaked data revealed the string:

```
heartbleedbelievethehype
```

Directory enumeration using Gobuster revealed several directories of interest:

- `/decode`
- `/encode`
- `/dev`

The `/decode` and `/encode` endpoints appeared to perform Base64 operations but did not execute server-side input.

Exploration of the `/dev` web directory revealed:

- `notes.txt` - describing poor security practices
- `hype_key` - a file containing hex-encoded data

The hex data was decoded into an RSA private key, which was protected using the SSH passphrase `heartbleedbelievethehype`.

---

## 6. Initial Access

Using the decoded RSA private key and SSH passphrase, the `hype` user account was accessed with SSH on port 22: 

```
ssh -i hype_key hype@10.129.232.136
```

This resulted in unprivileged access to the *Valentine* host.

---

## 7. Post-Exploitation

After authenticating as the user `hype`, local enumeration was performed:

- Ubuntu 12.04 LTS identified
- Kernel version 3.2.x

---

## 8. Privilege Escalation

Multiple exploits for Linux kernel and Ubuntu version were unsuccessful.

Further enumeration using `linpeas.sh` identified a suspicious `tmux` socket owned by root. By attaching to the socket, a root shell was obtained:

```
/usr/bin/tmux -S /.devs/dev_sess
```

This resulted in a root shell session.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/hype/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### Vulnerable OpenSSL Version (Heartbleed)

**Severity:** Critical

**Recommendation:** Upgrade OpenSSL to a non-vulnerable version and revoke all potentially compromised credentials.

### Sensitive Files Exposed via Web Server

**Severity:** High

**Recommendation:** Remove sensitive directories from web root and enforce proper access controls.

### Insecure Root tmux Session

**Severity:** Critical

**Recommendation:** Ensure privileged sessions do not expose world-accessible sockets and enforce strict permissions.

---

## 11. Appendix

### Resources Used

Heartbleed Proof-of-Concept (CVE-2014-0160):
[https://github.com/sensepost/heartbleed-poc](https://github.com/sensepost/heartbleed-poc)

linPEAS Privilege Escalation Script:
[https://github.com/peass-ng/PEASS-ng](https://github.com/peass-ng/PEASS-ng)