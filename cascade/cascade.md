# HTB Cascade - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-16

---

## 0. Lesson Learned

Read ldap enumeration results thoroughly for potential leaked passwords.

Always put `Administrator` in user list for CrackMapExec testing in case of password reuse.

---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 10.129.231.236
- **Domain:** cascade.local
- **Environment:** HackTheBox Cascade (Retired Machine)
- **Testing Window:** 2026-3-11 to 2016-3-16
- **Objective:** Complete domain compromise

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

Full TCP Port Scan:

```
nmap -p- -sCV 10.129.231.236 -T4 -oA cascade_tcp
```

Key services identified:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 135/tcp - RPC
- 139,445/tcp - SMB
- 389,636/tcp - LDAP
- 3268,3269/tcp - Global Catalog LDAP
- 5985/tcp - WinRM
- 49154-49165/tcp - RPC

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

**User Flag**: *REDACTED*

```
type C:\Users\s.smith\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full domain compromise.

---

## 10. Findings & Recommendations



---

## 11. Appendix

RPC Enumeration Commands -
[https://www.hackingarticles.in/active-directory-enumeration-rpcclient/](https://www.hackingarticles.in/active-directory-enumeration-rpcclient/)

Decrypting VNC Passwords - 
[https://github.com/billchaison/VNCDecrypt](https://github.com/billchaison/VNCDecrypt)

VNC Password Decryption One-Liner -

```
echo -n 'VNC_PASSWORD' | xxd -r -p  | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d -provider legacy -provider default | hexdump -Cv
```

AD Recycle Bin Group Membership Exploit -
[https://github.com/ivanversluis/pentest-hacktricks/blob/master/windows/active-directory-methodology/privileged-accounts-and-token-privileges.md](https://github.com/ivanversluis/pentest-hacktricks/blob/master/windows/active-directory-methodology/privileged-accounts-and-token-privileges.md)

AD Recycle Bin Exploit One-Liner -

```
Get-ADObject -filter 'isDeleted -eq $true' -includeDeletedObjects -Properties *
```