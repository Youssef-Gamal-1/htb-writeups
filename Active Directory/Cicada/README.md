# Penetration Test Report: Cicada.htb

## 1. Executive Summary
During the security assessment of the `cicada.htb` environment, a critical attack path was identified that allowed an unauthenticated attacker on the internal network to achieve full, unrestricted Domain Admin privileges. The compromise was made possible by a combination of insecure network file shares, unauthenticated user enumeration, cleartext credential leaks, over-privileged user groups, and the complete domain-wide disabling of anti-malware protections.

---

## 2. Attack Path Visual Map

```text
[Unauthenticated Attacker]
       │
       ▼ (Anonymous Read Access)
[Finding 1: HR Share / HR Note.txt] ───► Leaked New Hire Default Password
       │
       ▼ (Bypass Null Session Restriction)
[Finding 2: Active Directory User Enumeration via Guest Account]
       │
       ▼ (Password Spray via Validated Username List)
[Finding 3: Initial Access - michael.wrightson]
       │
       ▼ (LDAP Metadata / Description Scraping)
[Finding 4: Pivot to david.orelious Account]
       │
       ▼ (SMB Share Enumeration: \DEV\Backup_script.ps1)
[Finding 5: Pivot to emily.oscars (Backup Operator)]
       │
       ▼ (Registry Hive Export via SeBackupPrivilege)
[Finding 6: Offline Credential Dumping: SAM/SYSTEM Hives]
       │
       ▼ (Pass-the-Hash Attack)
[Full Domain Compromise: Administrator Account / Root Flag]
```

---

## 3. Detailed Technical Findings & Exploitation

### Finding 1: Plaintext Credential Exposure in Public HR Share
* **Severity**: Critical
* **Description**: The Domain Controller exposed an insecure network share named `HR` that allowed anonymous, unauthenticated read access. This share hosted internal operational documentation.
* **Exploitation**: An analysis of the accessible files revealed an onboarding document named `HR Note.txt`. This file explicitly leaked a standardized, highly complex corporate default password assigned to new hires:
  * **Default Password**: `Cicada$M6Corpb*@Lp#nZp!8`

### Finding 2: Broad Active Directory Information Leakage via Guest Account
* **Severity**: High
* **Description**: While modern Active Directory configurations strictly restrict anonymous null sessions, the built-in `Guest` account was left explicitly enabled with a blank password. This allowed network attackers to authenticate and issue Remote Procedure Calls (RPC) to the Security Account Manager (SAM) interface.
* **Exploitation**: The `netexec` tool was utilized to target the Domain Controller (`10.129.26.18`) by authenticating as `guest` and initiating Relative Identifier (RID) brute-forcing. This successfully leaked the complete domain user list:
  ```bash
  netexec smb 10.129.26.18 -u 'guest' -p '' --rid-brute
  ```

### Finding 3: Insecure New Hire Welcome Protocol & Password Spraying
* **Severity**: High
* **Description**: Because users were not forced to change their default credentials immediately upon initial account generation, an active window of exposure existed across the domain.
* **Exploitation**: By conducting a precise password spray using the default password recovered from `Finding 1` against the username list generated in `Finding 2`, initial access was established via the `michael.wrightson` account:
  ```bash
  netexec smb 10.129.26.18 -u user.txt -p 'Cicada\$M6Corpb*@Lp#nZp!8'
  ```

### Finding 4: Cleartext Credential Exposure within LDAP Descriptions
* **Severity**: Critical
* **Description**: Internal Active Directory account attributes contained highly sensitive operational data. Administrators had stored cleartext passwords directly inside user description metadata fields.
* **Exploitation**: Utilizing `ldapdomaindump` with the validated credentials of `michael.wrightson`, the directory structure was extracted offline. Parsing `domain_users.html` revealed a plaintext password string belonging to employee `david.orelious`:
  * **Username**: `david.orelious`
  * **Password**: `aRt\$Lp#7t*VQ!3`

### Finding 5: Plaintext Hardcoded Credentials within Configuration Scripts (\DEV Share)
* **Severity**: Critical
* **Description**: The authenticated profile of `michael.wrightson` possessed read permissions over a non-standard network share named `DEV`. This share hosted administrative deployment artifacts and internal backup automation scripts.
* **Exploitation**: Accessing the `DEV` share exposed a PowerShell automation file named `Backup_script.ps1`. Reviewing the code revealed hardcoded, plaintext operational credentials used to run background jobs:
  ```powershell
  \$username = "emily.oscars"
  \$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
  ```

### Finding 6: Domain-Wide Disabling of Anti-Malware Defenses (Windows Defender)
* **Severity**: Critical
* **Description**: Evaluation of Group Policy configurations (`Registry.pol` found within the environment) revealed a fatal defensive override rule explicitly pushing `DisableAntiSpyware` to the fleet.
* **Exploitation**: Windows Defender was completely disabled across the system. This allowed the execution of raw, un-obfuscated pentesting binaries, remote administration scripts, and data exfiltration techniques without triggering behavioral alerts or signature deletions.

### Finding 7: Over-Privileged Local Group Membership (Backup Operators Takeover)
* **Severity**: Critical
* **Description**: The compromised user account `emily.oscars` was found to be a member of the local **`Backup Operators`** and **`Remote Management Users`** groups. The Backup Operators group inherently possesses `SeBackupPrivilege`, allowing users to bypass system Access Control Lists (ACLs) to read or export restricted operating system files.
* **Exploitation**: 
  1. Remote session access was established via `evil-winrm` using Emily's credentials.
  2. Because standard snapshot tools (`ntdsutil`) were restricted by UAC integrity tokens, the local registry hives containing system secrets were exported directly using native system commands from inside `C:\temp`:
     ```powershell
     reg save HKLM\SAM c:\temp\SAM /y
     reg save HKLM\SYSTEM c:\temp\SYSTEM /y
     ```
  3. The hives were transferred offline and processed locally using `impacket-secretsdump` to extract the local `Administrator` NTLM hash:
     ```bash
     impacket-secretsdump -sam SAM -system SYSTEM local
     ```
  4. The resulting NTLM hash was leveraged in a Pass-the-Hash (PtH) maneuver via `evil-winrm` to achieve ultimate, administrative control over the Domain Controller and capture the root flag:
     ```bash
     evil-winrm -i 10.129.26.18 -u 'Administrator' -H '<EXTRACTED_NTLM_HASH>'
     ```

---

## 4. Strategic Remediation Plan

* **Secure Network Shares**: Immediately revoke unauthenticated anonymous read permissions on the `HR` share and enforce strict access control lists (ACLs) on the `DEV` share.
* **Disable the Guest Account**: Explicitly disable the built-in Active Directory `Guest` account across all Group Policies to stop anonymous user enumeration.
* **Enforce Password Expirations**: Mandate that all temporary default passwords assigned to new hires expire immediately upon creation, forcing a mandatory password change on the very first login attempt.
* **Purge Hardcoded Credentials**: Audit all script repositories, configuration files, and Active Directory description metadata fields. Ensure automated deployment and backup scripts do not store plaintext passwords.
* **Re-enable Endpoint Defenses**: Immediately revoke the Group Policy setting `DisableAntiSpyware`. Deploy centralized, modern Endpoint Detection and Response (EDR) agents to protect the SAM/SYSTEM registry paths.
* **Restrict Backup Operators**: Heavily restrict memberships within the `Backup Operators` group. Treat any user assigned to this group with the same tier of operational security as a Domain Administrator.
