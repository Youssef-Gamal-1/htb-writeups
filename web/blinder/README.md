# Blunder - Hack The Box

## Summary

Blunder was compromised through a vulnerable Bludit CMS installation. Version disclosure and leaked information enabled credential discovery, 
leading to administrative access and remote code execution. Further enumeration revealed another user's password hash, which was cracked to 
obtain user access. A sudo misconfiguration was then abused to gain root privileges.

## Enumeration

An Nmap scan revealed HTTP and FTP services. Web enumeration using Gobuster identified several interesting directories and files.

During enumeration:

* Bludit version **3.9.2** was identified through files within `bl-kernel` and `bl-content`.
* A `todo.txt` file was discovered containing the username **fergus**.

## Initial Access

A custom password wordlist was generated using Cewl against the website content.

The `/admin/login.php` page was brute-forced using the discovered username and generated wordlist, resulting in valid credentials:

* Username: fergus
* Password: RolandDeschain

After authenticating to the Bludit administration panel, the Metasploit module `linux/http/bludit_upload_images_exec` was used to achieve remote 
code execution and obtain a shell on the target.

## User Access

While enumerating the system, a password hash for user **hugo** was found in:

`/var/www/bludit-3.10.0a/bl-content/databases/users.txt`

The SHA1 hash was cracked successfully, allowing SSH access as hugo and retrieval of the user flag.

## Privilege Escalation

Reviewing sudo permissions with `sudo -l` revealed:

`(ALL, !root) /bin/bash`

This configuration was vulnerable to a known sudo bypass. Executing:

`sudo -u#-1 /bin/bash`

spawned a root shell, allowing retrieval of the root flag.

## Key Findings

1. Version disclosure enabled identification of a vulnerable CMS.
2. Sensitive information leakage exposed a valid username.
3. Weak credentials allowed administrative account compromise.
4. Password hashes were stored in accessible locations.
5. Misconfigured sudo permissions resulted in full privilege escalation.

## Lessons Learned

* Always inspect CMS-related files for version disclosure.
* Publicly accessible notes and backup files can reveal valuable information.
* Custom wordlists generated from target content can be highly effective.
* Local file enumeration often leads to credential reuse opportunities.
* Sudo misconfigurations should always be investigated thoroughly.
