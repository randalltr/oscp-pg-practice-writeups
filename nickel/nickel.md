# PGP Nickel - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-12

---

## 0. Lesson Learned

Always check C:\ for files.

---

## 1. Executive Summary

A penetration test was conducted against the Windows host *Nickel*. The objective was to identify vulnerabilities, obtain initial access, and escalate privileges to SYSTEM level. The host was compromised through web and service enumeration that uncovered a DevOps dashboard exposing backend functionality and Base64-encoded credentials, which provided SSH access as user *ariah*. Post-exploitation revealed a password-protected PDF containing infrastructure details. After cracking the password, SSH port forwarding exposed and internal API running as **NT AUTHORITY\SYSTEM** that allowed arbitrary PowerShell command execution. This was leveraged to add *ariah* to the local Administrators group, and SYSTEM-level access was obtained using PsExec, allowing retrieval of both user and administrator proof files.

---

## 2. Scope

- **Target:** 192.168.207.99
- **Environment:** Proving Grounds Practice Nickel
- **Testing Window:** 2026-2-11
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

A TCP scan was run with Nmap:

```
nmap -p- -sCV 192.168.207.99 -T4 -oA nickel_initial
```

Identifying the following open ports:

- 21/tcp - FTP (FileZilla ftpd 0.9.60 beta)
- 22/tcp - OpenSSH for Windows 8.1
- 135/tcp - MSRPC
- 139/tcp - NetBIOS
- 445/tcp - SMB
- 3389/tcp - RDP
- 8089/tcp - HTTP (Microsoft HTTPAPI 2.0)
- 33333/tcp - HTTP (Microsoft HTTPAPI 2.0)
- 5040/tcp, 7680/tcp - Unknown
- 49664-49669/tcp - RPC

An initial UDP scan was run with Nmap:

```
nmap -sU --top-ports 50 -T4 192.168.207.99 -oA nickel_udp
```

No significant UDP attack surface was identified.

---

## 5. Enumeration

---

## 6. Initial Access

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