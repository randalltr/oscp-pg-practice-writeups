# HTB Jeeves - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-19

---

## 0. Lesson Learned

If flags and files don't exist when they should, always run `dir /R` to check for hidden data attached to files as NTFS supports **alternate data streams**.

**CrackMapExec:** SMB/AD enum + credential validation + light command exec (legacy, still seen).

**NetExec:** Modern CME replacement; fast SMB/AD enum, cred spraying, tells you where creds work.

**Evil-WinRM:** PowerShell shell over WinRM; works with non-admin creds if allowed, clean and stable.

**PsExec:** SMB service-based exec; requires admin, drops you straight into SYSTEM.

---

## 1. Executive Summary

A penetration test was performed against the target host *Jeeves*. Multiple weaknesses were identified, including exposed administrative services, insecure credential storage, and improper handling of sensitive data. These issues enabled and attacker to gain initial access via a Jenkins service, extract stored credentials, reuse administrator hashes, and escalate privileges to SYSTEM. Full compromise of the host was achieved, including retrieval of user and root proof files.

---

## 2. Scope

- **Target:** 10.129.228.112
- **Environment:** HackTheBox Jeeves (Retired Machine)
- **Testing Window:** 2026-1-15 to 2026-1-16
- **Objective:** Obtain user-level and root-level access

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

An initial full TCP port scan was performed to identify exposed services.

```
nmap -p- -sCV 10.129.228.112 -T4 -oA jeeves_initial
```

This identified multiple open services, including HTTP (80), SMB (445), MSRPC (135), and a Jetty-based HTTP service on port 50000.

---

## 5. Enumeration

### Web Enumeration

Directory brute forcing was conducted against the HTTP services with both short and longer wordlists and looking for html extensions.

```
gobuster dir -u http://10.129.228.112 -w /usr/share/wordlists/dirb/common.txt --no-error

gobuster dir -u http://10.129.228.112:50000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html --no-error
```

This revealed an `/askjeeves` directory hosting a Jenkins dashboard.

### SMB Enumeration

SMB enumeration was attempted to identify accessible shares and authentication behavior.

```
smbclient -L //10.129.228.112 -N

smbclient -L //10.129.228.112 -U anonymous
```

SMB access was restricted, but weak security settings were later leveraged with valid credentials.

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