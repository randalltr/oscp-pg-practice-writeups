# HTB Valentine - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-14

---

## 0. Lesson Learned

Identifying the age of outdated Linux distributions helps find relevant vulnerabilities.

When privesc stalls, run linpeas and look for the red text with yellow background.

---

## 1. Executive Summary

A penetration test was conducted against the target host *Valentine*. The assessment identified multiple critical weaknesses related to outdated services and cryptographic misconfigurations. Exploitation of the **OpenSSL Heartbleed vulnerability (CVE-2014-0160)** resulted in credential disclosure, which enabled SSH access to the system. Further post-exploitation enumeration revealed a misconfigured `tmux` socket running as root, ultimately leading to full system compromise.

---

## 2. Scope

- **Target:** 10.129.232.136
- **Environment:** HackTheBox Valentine (Retired Machine)
- **Testing Window:** 2026-1-12 to 2026-1-14
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

---

## 5. Enumeration

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