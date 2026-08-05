# TryHackMe - Hacker Holidays 2026

# Day 5 - Beach Bar

**Category:** Web\
**Difficulty:** Medium

> *Write-up intentionally omits the flags.*

------------------------------------------------------------------------

## Challenge Overview

The objective was to assess a web application used by the fictional
Beach Bar. During enumeration an authenticated import feature accepted
YAML playlists. Further investigation revealed that the server parsed
user-controlled YAML using an unsafe loader, allowing arbitrary Python
object deserialization and ultimately remote code execution.

The goals were to:

-   Enumerate the application.
-   Identify the vulnerable import functionality.
-   Achieve command execution.
-   Enumerate the Linux host.
-   Obtain user access.
-   Escalate privileges through credential reuse.

------------------------------------------------------------------------

## Initial Enumeration

After authenticating to the application, the dashboard exposed several
features including an Import page capable of accepting YAML content
either through a text area or file upload.

The presence of YAML immediately suggested verifying how the application
parsed uploaded data.

------------------------------------------------------------------------

## Discovering the Vulnerability

Testing showed that the server accepted arbitrary YAML objects rather
than a strict playlist schema.

This behavior strongly indicated that an unsafe PyYAML loader was being
used.

Unsafe deserialization allows specially crafted YAML objects to
instantiate Python objects during parsing, which can execute system
commands.

------------------------------------------------------------------------

## Exploiting YAML Deserialization

A malicious YAML document was created using the Python object apply
syntax to invoke subprocess functions during deserialization.

Instead of loading playlist data, the application executed operating
system commands and returned their output inside the import page.

This confirmed successful Remote Code Execution.

------------------------------------------------------------------------

## Linux Enumeration

With command execution available, basic enumeration was performed to
understand the environment.

Useful actions included:

-   Identifying the current user.
-   Enumerating the filesystem.
-   Reading configuration files.
-   Searching for application files.
-   Inspecting the virtual environment.
-   Looking for credentials and reusable secrets.

Enumeration revealed application files, configuration data and
credentials that could be leveraged for further access.

------------------------------------------------------------------------

## User Access

Using the discovered command execution primitive, user-level files
became accessible.

The challenge user flag was recovered, confirming successful compromise
of the initial account.

**user flag:** THM{y4ml_pl4yl1st_pwns_th3_b34ch}

------------------------------------------------------------------------

## Privilege Escalation

Further enumeration uncovered credentials that were reused elsewhere on
the system.

Testing those credentials against the privileged account succeeded,
allowing escalation to root.

After switching users, the root directory became accessible and the
final challenge flag was retrieved.

**Root Flag:** THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}

------------------------------------------------------------------------

## Techniques Used

-   Web Enumeration
-   YAML Analysis
-   Unsafe Deserialization
-   Python Object Injection
-   Remote Code Execution
-   Linux Enumeration
-   Credential Discovery
-   Credential Reuse
-   Privilege Escalation

------------------------------------------------------------------------

## Lessons Learned

This challenge demonstrates why deserializing untrusted data with unsafe
loaders is extremely dangerous.

Applications should always use safe parsing functions when processing
YAML and should never allow arbitrary object construction from
user-controlled input.

The room also highlights how sensitive credentials stored on a host can
lead directly to privilege escalation when reused across accounts.

------------------------------------------------------------------------

## Skills Practiced

-   Web Application Testing
-   YAML Deserialization
-   Python Security
-   Linux Enumeration
-   Remote Code Execution
-   Privilege Escalation

------------------------------------------------------------------------

**Author:** Abdullah Sikandar
