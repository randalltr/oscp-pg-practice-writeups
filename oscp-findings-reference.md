## 1. Outdated or Vulnerable Software

### **Finding:** Outdated or End-of-Life Software

**Severity:** Critical

**Description:**
The target system was running outdated or unsupported software containing publicly known vulnerabilities.

**Impact:**
An attacker could exploit these vulnerabilities to execute arbitrary code and gain full system compromise.

**Recommendation:**
Upgrade to a supported version, apply all available security patches, and implement a formal patch management process.

### **Finding:** Vulnerable Web Application Allowing Remote Code Execution

**Severity:** Critical

**Description:**
A publicly accessible web application contained a known vulnerability that allowed arbitrary command execution.

**Impact:**
Unauthenticated attackers could gain system-level access to the host.

**Recommendation:**
Remove or upgrade the vulnerable application, restrict administrative interfaces, and isolate legacy services.

---

## 2. Weak or Exposed Credentials

### **Finding:** Hardcoded Credentials in Source Code

**Severity:** High

**Description:**
Sensitive credentials were embedded directly within application source code accessible to low-privileged users.

**Impact:**
Attackers could retrieve credentials and use them to gain unauthorized access to services or escalate privileges.

**Recommendation:**
Remove hardcoded credentials from source code, use secure configuration storage or secret management solutions, and restrict access to sensitive files.

### **Finding:** Credentials Stored in Registry or System Files (Windows)

**Severity:** High

**Description:**
Sensitive credentials were stored in the Windows registry or system configuration files accessible to low-privileged users.

**Impact:**
Attackers could extract credentials and escalate privileges or move laterally.

**Recommendation:**
Avoid storing plaintext credentials in the registry or system files. Use secure credential storage mechanisms and restrict access permissions.

### **Finding:** Default or Weak Credentials

**Severity:** High

**Description:**
The application used weak or guessable credentials.

**Impact:**
Attackers could gain unauthorized access to administrative interfaces or services.

**Recommendation:**
Enforce strong password policies, implement account lockout mechanisms, and require unique credentials per service.

### **Finding:** Credential Exposure in Files or Web Content

**Severity:** Critical

**Description:**
Sensitive credentials were stored in web-accessible files or user-readable locations.

**Impact:**
Attackers could retrieve credentials and escalate privileges.

**Recommendation:**
Remove sensitive files from web directories, restrict file permissions, and use centralized secret management solutions.

### **Finding:** Credential Reuse Across Services

**Severity:** High

**Description:**
The same credentials were used across multiple services.

**Impact:**
Compromise of one service could lead to compromise of additional services.

**Recommendation:**
Enforce unique credentials per service and implement credential rotation policies.

### **Finding:** Credentials Stored in Configuration Files

**Severity:** High

**Description:**
Service or application credentials were stored in plaintext configuration files accessible to low-privileged users.

**Impact:**
Attackers could retrieve credentials and reuse them to access other services or escalate privileges.

**Recommendation:**
Store credentials securely using protected configuration storage or secret management systems and restrict file permissions.

### **Finding:** Password Recovered from Local History or Scripts

**Severity:** High

**Description:**
Sensitive credentials were found in shell history, scripts, or user-accessible files.

**Impact:**
Attackers could use the credentials to escalate privileges or access other systems.

**Recommendation:**
Avoid storing credentials in plaintext. Sanitize scripts, clear history files, and enforce strict file permissions.

---

## 3. Unauthenticated or Misconfigured Network Services

### **Finding:** Anonymous SMB Share Access

**Severity:** Critical

**Description:**
SMB shares were accessible without authentication, allowing anonymous enumeration of files and directories.

**Impact:**
Attackers could access sensitive data, identify credential files, or discover information useful for further compromise.

**Recommendation:**
Disable null sessions, restrict anonymous access to SMB shares, and enforce authentication for all file share access.

### **Finding:** Unrestricted SMB Share Access

**Severity:** High

**Description:**
An SMB share was accessible without authentication or with overly permissive access controls.

**Impact:**
Attackers could read or modify sensitive files, potentially leading to credential exposure or privilege escalation.

**Recommendation:**
Restrict SMB shares to authenticated users, apply least-privilege permissions, and disable guest or anonymous access.

### **Finding:** Unrestricted NFS Share

**Severity:** Critical

**Description:**
An NFS share was accessible without proper access restrictions.

**Impact:**
Attackers could mount the share, access sensitive files, or introduce malicious content to escalate privileges.

**Recommendation:**
Restrict NFS access to trusted hosts, enforce proper export permissions, and apply root-squash where appropriate.

### **Finding:** Insecure SNMP Configuration

**Severity:** Medium

**Description:**
SNMP was accessible using default or weak community strings.

**Impact:**
Attackers could enumerate system information, users, and services.

**Recommendation:**
Disable SNMP if not required or enforce strong community strings and restrict access via firewall rules.

### **Finding:** Unauthenticated Service Exposure

**Severity:** Critical

**Description:**
A network service was accessible without authentication.

**Impact:**
Attackers could directly execute commands or access sensitive data.

**Recommendation:**
Restrict service access to localhost or trusted networks, enforce authentication, and apply firewall rules.

### **Finding:** Anonymous File Transfer Access

**Severity:** Critical

**Description:**
Anonymous login allowed file uploads into a web-accessible directory.

**Impact:**
Attackers could upload malicious files and gain remote code execution.

**Recommendation:**
Disable anonymous access, separate file transfer directories from web roots, and monitor file transfer activity.

---

## 4. Web Application Vulnerabilities

### **Finding:** Broken Access Control in API Endpoint

**Severity:** High

**Description:**
An API endpoint lacked proper authorization checks, allowing unauthorized users to access restricted functionality.

**Impact:**
Attackers could perform privileged actions or access sensitive data without proper authentication.

**Recommendation:**
Implement proper access control checks on all API endpoints and enforce role-based access control.

### **Finding:** Insecure File Permissions on Web Root

**Severity:** High

**Description:**
The web root directory or application files were writable by a low-privileged user.

**Impact:**
Attackers could modify web content or upload malicious scripts, leading to remote code execution.

**Recommendation:**
Restrict write permissions on web directories and enforce least-privilege access controls.

### **Finding:** Backup or Archive Files Exposed on Web Server

**Severity:** High

**Description:**
Backup or archive files containing sensitive data were accessible through the web server.

**Impact:**
Attackers could retrieve credentials, source code, or configuration files.

**Recommendation:**
Remove backup files from web-accessible directories and enforce strict access controls.

### **Finding:** Information Disclosure via Directory Listing

**Severity:** Medium

**Description:**
Directory listing was enabled on a web server, exposing sensitive files and internal structure.

**Impact:**
Attackers could identify sensitive files, credentials, or application components.

**Recommendation:**
Disable directory listing and restrict access to sensitive directories.

### **Finding:** Hardcoded Credentials in Web Application

**Severity:** High

**Description:**
Credentials were embedded directly within application code or client-side responses.

**Impact:**
Attackers could extract credentials and gain unauthorized access.

**Recommendation:**
Remove hardcoded credentials and use secure configuration or secret management solutions.

### **Finding:** Arbitrary File Upload

**Severity:** Critical

**Description:**
The web application allowed uploading executable files.

**Impact:**
Attackers could gain remote code execution.

**Recommendation:**
Enforce strict file-type validation, disable execution in upload directories, and apply secure coding practices.

### **Finding:** Local or Remote File Inclusion

**Severity:** Critical

**Description:**
User-controlled input was used in file inclusion operations.

**Impact:**
Attackers could read sensitive files or execute code.

**Recommendation:**
Validate all file paths, implement allowlists, and disable dynamic inclusion where unnecessary.

### **Finding:** Directory Traversal

**Severity:** Critical

**Description:**
User input allowed access to unintended file paths.

**Impact:**
Sensitive system files could be disclosed.

**Recommendation:**
Sanitize user input, enforce fixed directory paths, and reject traversal sequences.

### **Finding:** Command Injection

**Severity:** Critical

**Description:**
User-controlled input was executed as system commands.

**Impact:**
Attackers could execute arbitrary commands on the system.

**Recommendation:**
Validate and sanitize inputs, avoid direct shell execution, and use parameterized commands.

---

## 5. Misconfigured Privileges and Sudo Rights

### **Finding:** Improperly Configured SUID Binary Allowing Arbitrary Backups

**Severity:** Critical

**Description:**
A SUID binary allowed low-privileged users to perform backup operations with elevated privileges, enabling arbitrary command execution.

**Impact:**
Attackers could exploit the SUID binary to escalate privileges to root.

**Recommendation:**
Remove unnecessary SUID permissions, restrict privileged binaries, and enforce least-privilege principles.

### **Finding:** Insecure Command Execution via MongoDB Scheduler

**Severity:** Critical

**Description:**
A scheduled task executed user-controlled commands through a MongoDB-backed scheduler, allowing arbitrary command execution with elevated privileges.

**Impact:**
Attackers could execute arbitrary commands and escalate privileges to root or SYSTEM.

**Recommendation:**
Sanitize and validate all scheduler commands, restrict MongoDB access, and enforce least-privilege execution for scheduled tasks.

### **Finding:** Sudo Rights on Dangerous Binaries

**Severity:** Critical

**Description:**
A user was permitted to execute dangerous binaries via sudo that allowed shell escape or arbitrary command execution.

**Impact:**
Attackers could escalate privileges to root.

**Recommendation:**
Restrict sudo access to only required binaries and avoid allowing interactive or shell-capable programs.

### **Finding:** SUID Binary Allowing Privilege Escalation

**Severity:** Critical

**Description:**
A SUID binary was present that allowed execution of commands with elevated privileges.

**Impact:**
Attackers could exploit the binary to escalate privileges to root.

**Recommendation:**
Remove unnecessary SUID permissions and audit privileged binaries regularly.

### **Finding:** PATH Environment Variable Abuse

**Severity:** High

**Description:**
A privileged script or binary relied on the system PATH and executed commands without absolute paths.

**Impact:**
Attackers could place malicious binaries in writable directories to escalate privileges.

**Recommendation:**
Use absolute paths in privileged scripts and restrict write access to directories in the PATH.

### **Finding:** Misconfigured Sudo Permissions

**Severity:** Critical

**Description:**
A user was allowed to execute privileged commands via sudo without a password.

**Impact:**
Attackers could escalate privileges to root.

**Recommendation:**
Remove unnecessary NOPASSWD entries and enforce least-privilege sudo policies.

### **Finding:** Excessive Token or Service Privileges

**Severity:** High

**Description:**
A service account possessed high-risk privileges such as token impersonation rights.

**Impact:**
Attackers could escalate privileges through token impersonation.

**Recommendation:**
Restrict high-risk privileges and assign only required permissions to service accounts.

---

## 6. Writable Files, Services, or Scheduled Tasks

### **Finding:** Writable Configuration File Used by Privileged Service

**Severity:** Critical

**Description:**
A configuration file used by a privileged service was writable by a low-privileged user.

**Impact:**
Attackers could modify the configuration to execute malicious commands with elevated privileges.

**Recommendation:**
Restrict write permissions on service configuration files and enforce least-privilege access.

### **Finding:** Writable Service Executable

**Severity:** Critical

**Description:**
A service executable or configuration was writable by a low-privileged user.

**Impact:**
Attackers could replace the executable and gain elevated privileges.

**Recommendation:**
Restrict write permissions on service binaries and enforce least-privilege access controls.

### **Finding:** Writable Script Executed by Root

**Severity:** Critical

**Description:**
A user had write access to a script executed by a privileged account.

**Impact:**
Attackers could inject commands and gain root access.

**Recommendation:**
Ensure privileged scripts are owned by root, remove write permissions for other users, and monitor file integrity.

### **Finding:** Writable Scheduled Task or Service Binary

**Severity:** Critical

**Description:**
A scheduled task or service executed files from writable directories.

**Impact:**
Attackers could replace binaries and escalate privileges.

**Recommendation:**
Restrict write access to service paths, apply least-privilege permissions, and monitor scheduled tasks.

---

## 7. Kernel or OS Privilege Escalation

### **Finding:** Unquoted Service Path

**Severity:** High

**Description:**
A Windows service used an unquoted service path containing spaces.

**Impact:**
Attackers could place malicious executables in the path and gain SYSTEM privileges.

**Recommendation:**
Quote all service paths and restrict write permissions on service directories.

### **Finding:** Misconfigured Windows Service Permissions

**Severity:** High

**Description:**
A Windows service allowed modification by low-privileged users.

**Impact:**
Attackers could alter service configuration to execute code as SYSTEM.

**Recommendation:**
Restrict service permissions and audit service configurations regularly.

### **Finding:** Local Kernel Exploit

**Severity:** High

**Description:**
The operating system contained a vulnerable kernel component.

**Impact:**
Attackers could escalate privileges to root or SYSTEM.

**Recommendation:**
Apply all security patches, upgrade to a supported OS version, and implement a patch management program.

---

## 8. Insecure Development or Administrative Interfaces

### **Finding:** Exposed Development or Administrative Interface

**Severity:** High

**Description:**
A development or administrative interface was publicly accessible.

**Impact:**
Attackers could execute commands or access sensitive functionality.

**Recommendation:**
Remove development tools from production, require authentication, and restrict access via network controls.

---

## 9. Insecure Internal Service Exposure

### **Finding:** Internal Service Accessible via Port Forwarding

**Severity:** High

**Description:**
Internal services were accessible through SSH tunneling or port forwarding.

**Impact:**
Attackers could reach privileged services and escalate access.

**Recommendation:**
Restrict SSH port forwarding, enforce access controls, and monitor tunneling activity.

---

## 10. Weak SMB or NTLM Security

### **Finding:** SMBv1 Enabled

**Severity:** High

**Description:**
The system supported the legacy SMBv1 protocol.

**Impact:**
SMBv1 is vulnerable to multiple remote code execution and relay attacks.

**Recommendation:**
Disable SMBv1 and enforce modern SMB protocols with signing enabled.

### **Finding:** Weak SMB or NTLM Configuration

**Severity:** High

**Description:**
SMB signing or secure authentication was not enforced.

**Impact:**
Attackers could perform relay or pass-the-hash attacks.

**Recommendation:**
Enforce SMB signing, disable NTLM where possible, and implement credential protection mechanisms.

---

## 11. Sensitive Data Exposure

### **Finding:** Group Policy Preference Password Exposure

**Severity:** Critical

**Description:**
Group Policy Preferences stored credentials using a reversible encryption method with a publicly known key, allowing password recovery.

**Impact:**
Attackers could extract and decrypt stored credentials, potentially gaining administrative access to the domain.

**Recommendation:**
Remove all Group Policy Preference passwords, implement LAPS or gMSA accounts, and audit SYSVOL and replication permissions.

### **Finding:** Sensitive Data Stored Insecurely

**Severity:** High

**Description:**
Sensitive files or credentials were stored in accessible locations.

**Impact:**
Attackers could retrieve confidential data and escalate privileges.

**Recommendation:**
Restrict file permissions, use centralized secret storage, and conduct regular audits.

---

## 12. Active Directory Weaknesses

### **Finding:** Kerberoastable Administrator Account

**Severity:** Critical

**Description:**
A highly privileged account had a Service Principal Name (SPN) configured, making it vulnerable to Kerberoasting attacks.

**Impact:**
Attackers could request service tickets, crack the password offline, and gain domain administrator privileges.

**Recommendation:**
Remove SPNs from privileged accounts, use dedicated service accounts, and enforce strong, random passwords.

### **Finding:** Weak Domain Administrator Password

**Severity:** Critical

**Description:**
A domain administrator account used a weak password that was susceptible to offline cracking.

**Impact:**
Attackers could recover the password and gain full control of the domain.

**Recommendation:**
Enforce long, random passwords for privileged accounts, monitor Event ID 4769 for suspicious ticket activity, and implement regular password rotation policies.

### **Finding:** Kerberoastable Service Account

**Severity:** High

**Description:**
A service account had a Service Principal Name (SPN) and could be targeted for Kerberoasting.

**Impact:**
Attackers could extract and crack service account passwords offline.

**Recommendation:**
Use strong, complex passwords for service accounts and rotate them regularly.

### **Finding:** AS-REP Roastable User Account

**Severity:** High

**Description:**
A user account did not require Kerberos preauthentication.

**Impact:**
Attackers could retrieve and crack authentication material offline.

**Recommendation:**
Enable Kerberos preauthentication and enforce strong password policies.

### **Finding:** Excessive Domain Privileges

**Severity:** Critical

**Description:**
A domain account possessed excessive privileges allowing lateral movement or domain compromise.

**Impact:**
Attackers could escalate privileges across the domain.

**Recommendation:**
Apply least-privilege principles and regularly audit group memberships.

---

## 13. Scheduled Task and Service Misconfigurations

### **Finding:** Misconfigured Scheduled Task

**Severity:** Critical

**Description:**
A scheduled task executed with elevated privileges was modifiable by a low-privileged user.

**Impact:**
Attackers could alter the task to execute malicious commands and gain elevated privileges.

**Recommendation:**
Restrict write permissions on scheduled tasks and enforce least-privilege access controls.

### **Finding:** Service Running as Root or SYSTEM with Excessive Permissions

**Severity:** High

**Description:**
A service running with elevated privileges had insecure file or directory permissions.

**Impact:**
Attackers could modify service files and escalate privileges.

**Recommendation:**
Restrict service file permissions and run services with the minimum required privileges.
---

## Ultra-Short Emergency Exam Reference (1-Line Fixes)

- Outdated software → Patch or upgrade.
- Default credentials → Enforce strong passwords.
- Credential exposure → Remove from web paths.
- Anonymous FTP → Disable anonymous access.
- File upload → Validate files, disable execution.
- LFI/RFI → Sanitize input, use allowlists.
- Command injection → Validate input, avoid shell.
- Sudo misconfig → Remove NOPASSWD.
- Writable cron/service → Restrict permissions.
- Kernel exploit → Patch OS.
- Exposed admin interface → Restrict access.
- Unauthenticated service → Require authentication.
- Weak SMB/NTLM → Enforce SMB signing, disable NTLM.

---

## Highest-Probability OSCP Additions (Top 10)

If you only add a few, prioritize these:

- SUID privilege escalation
- Writable service binary
- PATH hijacking
- Credentials in config files
- NFS misconfiguration
- SNMP default community string
- Directory listing disclosure
- Kerberoasting
- AS-REP roasting
- Windows service permission abuse

These appear in a large percentage of OSCP-style machines.