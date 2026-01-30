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