# Intermediate Nmap — TryHackMe

**Platform:** TryHackMe **Difficulty:** Intermediate **Category:** Network Scanning & Enumeration **Type:** Quick CTF

## **1. Reconnaissance**

Initial Nmap scan:

```bash
nmap -sC -sV -Pn -T4 <TARGET_IP> -oN nmap.txt
```

### **Open Ports**

|**Port**|**Service**|**Finding**|
|---|---|---|
|22/tcp|SSH|OpenSSH 8.2p1|
|2222/tcp|SSH|OpenSSH 8.2p1|
|31337/tcp|Unknown|Exposed credentials|

## **2. Enumeration**

The service running on port `31337/tcp` returned the following message:

```text
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

The service exposed credentials in plaintext.

**Username:** `ubuntu`

## **3. Initial Access**

Used the discovered credentials to connect via SSH:

```bash
ssh ubuntu@<TARGET_IP>
```

Verified the current user:

```bash
whoami
id
```

The output confirmed access as the `ubuntu` user.

## **4. Flag Discovery**

Enumerated the home directories and located the flag:

```bash
cd /home
ls
cd user
cat flag.txt
```

**Flag:**

```text
flag{251f309497a18888dde5222761ea88e4}
```

## **5. Vulnerability Summary**

**Finding:** Plaintext Credential Disclosure  
**Affected Port:** TCP/31337  
**Impact:** Exposed credentials enabled SSH access to the target system.

### **Remediation**

- Never expose credentials in service responses.
- Remove hardcoded passwords from application configurations.
- Rotate credentials that have been exposed.
- Restrict access to unnecessary services and ports.

## **6. Key Takeaways**

- Nmap service detection can reveal information beyond service versions.
- Unidentified services running on unusual ports deserve further investigation.
- Information disclosure can provide a direct initial access vector.
- Basic enumeration after gaining access can quickly reveal accessible flags.