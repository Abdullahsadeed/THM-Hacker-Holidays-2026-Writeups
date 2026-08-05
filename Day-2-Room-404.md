# TryHackMe Hacker Holidays 2026 - Room 404 Write-up

## Challenge Information

- **Platform:** TryHackMe
- **Event:** Hacker Holidays 2026
- **Room:** Room 404
- **Category:** Web
- **Difficulty:** Very Easy
- **Points:** 30

---

## Description

Byte Lotus Hotel launched its guest-experience platform quickly, but the developers accidentally exposed more than the intended website.

The goal of this challenge was to discover hidden resources, recover exposed source code, and find the flag.

---

# Enumeration

The challenge provided a web application running on port 8080.

I started by visiting:

```
http://MACHINE_IP:8080
```

The homepage only contained basic hotel information and did not reveal anything interesting.

Since the challenge category was Web and the description mentioned:

> "The rooms it never lists are the ones worth finding."

I started directory enumeration.

---

# Directory Enumeration

I used Gobuster to search for hidden directories and files:

```bash
gobuster dir -u http://MACHINE_IP:8080 -w /usr/share/wordlists/dirb/common.txt
```

The scan revealed:

```
/.git/HEAD
/app.js
```

The exposed `.git` directory was the important discovery.

A publicly accessible Git repository can reveal:

- Application source code
- Commit history
- Deleted files
- Developer information
- Sensitive data

---

# Exploiting the Exposed Git Repository

I confirmed that the Git repository was accessible:

```bash
curl http://MACHINE_IP:8080/.git/HEAD
```

The server returned Git information, confirming that the repository was exposed.

Normally tools such as `git-dumper` can be used to download exposed repositories, but instead I manually downloaded the repository using wget.

Command used:

```bash
wget -r -np -nH http://MACHINE_IP:8080/.git/
```

This downloaded the Git metadata, including:

```
.git/
├── HEAD
├── config
├── index
├── logs/
├── objects/
└── refs/
```

---

# Recovering the Source Code

After downloading the Git repository files, I analyzed the repository structure.

The exposed Git data contained the commit information required to restore the application's source.

I initialized the repository:

```bash
git init
```

Then checked the available commit history:

```bash
git log
```

The commit history revealed the hidden source information.

---

# Finding the Flag

After recovering and analyzing the source code, the hidden flag was discovered.

```
[FLAG REMOVED]
```

---

# Vulnerability

## Exposed Git Repository

The vulnerability was caused by leaving the `.git` directory accessible from the web server.

This is a serious information disclosure issue because attackers can recover internal application files and development history.

---

# Impact

An attacker could potentially:

- Read application source code
- Discover hidden endpoints
- Recover deleted files
- Find credentials or secrets
- Understand application logic

---

# Remediation

To prevent this vulnerability:

- Block access to hidden files and directories.
- Remove development files before deployment.
- Use secure deployment pipelines.
- Configure web servers to deny access to `.git`.
- Perform security reviews before publishing applications.

Example:

```
location ~ /\.git {
    deny all;
}
```

---

# Tools Used

- Gobuster
- wget
- Git
- curl

---

# Attack Chain

```
Web Application
        |
        v
Directory Enumeration
        |
        v
Exposed /.git Directory
        |
        v
Git Repository Recovery
        |
        v
Source Code Disclosure
        |
        v
Flag Discovery
```

---

# Key Takeaways

This challenge demonstrated that security issues are not always caused by complicated exploits.

A simple deployment mistake, such as exposing a Git repository, can reveal sensitive application information.

Proper enumeration and awareness of common misconfigurations are essential skills in web security.