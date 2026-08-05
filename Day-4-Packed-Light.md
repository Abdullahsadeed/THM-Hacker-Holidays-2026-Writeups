# TryHackMe - Hacker Holidays 2026
# Day 4 - Packed Light

**Category:** Forensics
**Difficulty:** Easy

---

## Challenge Overview

The challenge provided a network capture from the Byte Lotus Hotel guest network. The briefing suggested that data was being quietly exfiltrated through seemingly normal traffic.

The objective was to:

- Analyze the PCAP.
- Identify the covert communication channel.
- Recover the hidden data.
- Decode it to obtain the flag.

---

## Initial Analysis

The provided capture was opened in **Wireshark**.

Several HTTP requests immediately stood out because they were sent at extremely regular intervals to a service running on port **8080**, matching the clue given in the challenge description.

The repeated requests suggested automated communication rather than normal user activity.

---

## Investigating the HTTP Traffic

Following the HTTP streams revealed repeated requests containing an unusual cookie value.

Instead of behaving like a normal session identifier, the cookie changed with every request while maintaining a consistent structure.

This strongly indicated it was being used as a covert data channel.

---

## Finding the Source

Inspection of the downloaded resources showed a Python script responsible for generating the requests.

The script performed three operations:

1. Captured data.
2. XOR encrypted the captured bytes using a hardcoded key.
3. Base64 encoded the encrypted result.
4. Sent the encoded output inside the HTTP Cookie header.

This explained why the traffic appeared legitimate while secretly transmitting data.

---

## Recovering the Data

The recovery process consisted of:

- Collecting every encoded cookie value in chronological order.
- Base64 decoding each value.
- Reversing the XOR encryption using the same key embedded in the script.
- Concatenating the recovered plaintext.

The reconstructed output revealed the hidden message containing the challenge flag.

---

## Techniques Used

- Wireshark
- HTTP Stream Analysis
- Traffic Pattern Analysis
- Base64 Decoding
- XOR Decryption
- Static Script Analysis

---

## Lessons Learned

This challenge demonstrates how attackers can hide information inside completely legitimate protocols.

Rather than creating suspicious outbound connections, malware can abuse common HTTP headers such as Cookies, making the traffic appear normal at first glance.

Even simple encryption combined with standard encoding can effectively conceal stolen information unless the underlying protocol is carefully inspected.

---

## Skills Practiced

- Network Forensics
- PCAP Analysis
- HTTP Analysis
- Covert Channel Detection
- Base64 Analysis
- XOR Encryption Analysis

**🚩Flag:** THM{V3r4_1s_w4tch1ng_0veR_y0u}