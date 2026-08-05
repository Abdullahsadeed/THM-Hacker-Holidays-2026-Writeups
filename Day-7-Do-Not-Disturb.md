# Do Not Disturb - TryHackMe Write-up

> **Platform:** TryHackMe  
> **Room:** Do Not Disturb  
> **Category:** Web Exploitation  
> **Difficulty:** Medium

---

# Overview

In this room, the objective was to analyze a vulnerable Node.js web application, identify a Server-Side Template Injection (SSTI) vulnerability, achieve Remote Code Execution (RCE), and retrieve the user flag.

---

# Enumeration

After deploying the machine, I visited the target application.

The landing page presented a hotel-themed login portal named **Byte Lotus**.

## Observations

- Login functionality
- Staff portal
- Session-based authentication
- Express.js backend
- EJS template engine

---

# Source Code Analysis

Reviewing the application source revealed the following code responsible for rendering user templates:

```javascript
rendered = ejs.render(template, {
    guest: req.session.user.username,
    hotel: 'Byte Lotus'
});
```

The application rendered user-controlled input directly using **EJS**, indicating a possible **Server-Side Template Injection (SSTI)** vulnerability.

---

# Identifying SSTI

To verify template injection, a simple payload was supplied.

```ejs
<%= process.version %>
```

The payload executed successfully, confirming arbitrary JavaScript execution within the application's Node.js process.

---

# Remote Code Execution

Since arbitrary JavaScript execution was possible, Node.js modules could be accessed using:

```javascript
process.mainModule.require(...)
```

This allowed interaction with built-in modules such as:

- `child_process`
- `fs`

Resulting in full Remote Code Execution (RCE).

---

# Post Exploitation

After obtaining RCE, the next step was to enumerate the environment.

The following information was gathered:

- Current working directory
- Environment variables
- Directory structure
- Accessible files

Using the built-in **fs** module was more reliable than relying solely on shell commands.

Example:

```ejs
<%= process.mainModule.require('fs').readdirSync('/').join('\n') %>
```

---

# Retrieving the User Flag

After locating the correct directory, the user flag was successfully read from the target machine, completing the room.

**🚩Flag:** THM{w4rm_s3ss10n_h1j4ck3d}

---

# Skills Practiced

- Web Application Enumeration
- Source Code Analysis
- Express.js Security
- EJS Template Injection
- Server-Side Template Injection (SSTI)
- Remote Code Execution (RCE)
- Post-Exploitation Enumeration
- Node.js Internals

---

# Mitigation

Developers can prevent this vulnerability by:

- Never rendering user-controlled templates.
- Avoiding direct use of `ejs.render()` with untrusted input.
- Sanitizing and validating all user input.
- Following the Principle of Least Privilege.
- Restricting application access to sensitive files.

---

## Privilege Escalation

After doing reverse shell i successfully gained root flag.

**Root Flag:** THM{r4w_d1sk_4cc3ss_w4s_t00_much}

---

# Key Takeaways

This room demonstrated how a seemingly harmless template engine can become a critical vulnerability when user-controlled input is rendered without proper validation. It also reinforced the importance of careful enumeration after achieving initial code execution.

---

## Conclusion

**Do Not Disturb** is an excellent practical room for understanding **Server-Side Template Injection (SSTI)** in Node.js applications. It combines source code review, web exploitation, and post-exploitation techniques, making it a valuable exercise for anyone learning offensive web security.

---