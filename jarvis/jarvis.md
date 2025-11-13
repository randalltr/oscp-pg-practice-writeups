# HTB Jarvis - OSCP-Style Penetration Test Report
**Author:** RandallTr
**Target:** 10.129.229.137
**Environment:** HTB (Retired Machine)
**Date:** 2025-11-13

---

## 1. High-Level Summary
A penetration test was conducted against the *Jarvis* host on the HackTheBox network. Multiple critical vulnerabilities were discovererd, including:
- SQL Injection in the `/room.php?cod=` parameter.
- Plaintest database credentials stored on the web server.
- Weak database configuration allowing SQL file reads via `LOAD_FILE()`.
- Outdated phpMyAdmin version vulnerable to authenticated LFI -> RCE.
- Privilege escalation through misconfigured `sudo` fules and a SUID `systemctl` binary.

Successful exploitation yielded:
- Remote code execution as `www-data`.
- Privilege escalation to the user `pepper`.
- Full root compromise of hte system.

All flags were retrieved.

---

