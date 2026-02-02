# HTB Poison - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-2 (GROUNDHOG DAY!!!)

---

## 0. Lesson Learned

Always check running processes.

---

## 1. Executive Summary

An external penetration test was conducted against the target system *Poison*. The assessment resulted in full system compromise, including root-level access, by chaining multiple weaknesses: information disclosure via a vulnerable web application, credential exposure, and insecure internal service exposure abused through SSH port forwarding.

---

## 2. Scope

- **Target:** 10.129.1.254
- **Environment:** HackTheBox Poison (Retired Machine)
- **Testing Window:** 2026-1-31
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

A full TCP port scan was conducted with Nmap:

```
nmap -p- -sCV 10.129.1.254 -T4 -oA poison_initial
```

Open services identified:

- 22/tcp - OpenSSH 7.2 (FreeBSD)
- 80/tcp - Apache httpd 2.4.29 with PHP 5.6.32

---

## 5. Enumeration

### Web Enumeration

PHP testing files were discovered on the web server:

- `info.php`
- `phpinfo.php`
- `ini.php`
- `listfiles.php`

### Sensitive File Disclosure

Accessing `listfiles.php` revealed `pwdbackup.txt`, containing a password encoded multiple times in Base64:

```
http://10.129.1.254/pwdbackup.txt
```

Using CyberChef to Base64 decode this text file 13 times revealed the password `Charix!2#4%6&8(0`.

---

## 6. Initial Access

Recovered credentials were used to authenticate over SSH port 22 as user `charix`:

```
ssh charix@10.129.1.254
```

Successful user-level access as obtained.

---

## 7. Post Exploitation

A `secret.zip` file was located in `/home/charix`. Attempted extraction on the remote machine revealed password protection:

```
unzip secret.zip
```

The `secret.zip` file was copied to the attacker machine using the disclosed password:

```
scp charix@10.129.1.154:/home/charix/secret.zip .
```

Then the `secret.zip` file was extracted using the reused password:

```
unzip secret.zip
```

This produced a file named `secret`, later identified as a VNC password file.

---

## 8. Privilege Escalation

Process inspection identified a root-owned VNC service bound to localhost:

```
ps aux
```

Relevant process:

- `Xvnc` running as root on `127.0.0.1:5901`

### SSH Port Forwarding

SSH tunneling was used to access the internal VNC service:

```
ssh -L 5904:localhost:5901 charix@10.129.1.254
```

### VNC Connection

A VNC session was initiated by using the recovered password file:

```
vncviewer -passwd secret localhost:5904
```

This resulted in root desktop access.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/charix/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### Sensitive Credential Exposure via Web-Accessible Backup File

**Severity:** Critical

**Recommendation:** Remove backup and credential files from web-accessible directories. Enforce strict access controls on sensitive files and conduct regular reviews to ensure no confidential data is exposed through the web server.

### Insecure Local File Inclusion (LFI) in Web Application

**Severity:** Critical

**Recommendation:** Validate and sanitize all user-supplied input used in file inclusion functions. Implement strict allowlists for permitted files and disable dynamic file inclusion where not required.

### Credential Reuse Across Services

**Severity:** High

**Recommendation:** Enforce unique credentials per service and implement credential rotation policies. Compromised web credentials should never grant access to system-level services such as SSH.

### Insecure Internal Service Exposure (VNC)

**Severity:** Critical

**Recommendation:** Disable unnecessary services such as VNC on production systems. If required, restrict access via authentication, firewall rules, and binding to restricted interfaces with additional access controls.

### Abuse of SSH Port Forwarding to Access Privileged Services

**Severity:** High

**Recommendation:** Restrict SSH port forwarding where not operationally required. Monitor and log SSH tunneling activity and apply role-based access controls to prevent lateral or privilege escalation.

### Storage of Privileged Service Credentials in User-Readable Files

**Severity:** Critical

**Recommendation:** Do not store privileged service credentials in user-accessible locations. Secure sensitive authentication material using proper permission restrictions or centralized secrets management solutions.


