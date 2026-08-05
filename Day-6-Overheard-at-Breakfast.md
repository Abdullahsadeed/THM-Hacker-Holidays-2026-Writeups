# Overheard at Breakfast

**Category:** OSINT  
**Difficulty:** Easy  

## Challenge Description

Two strangers are having breakfast at the Byte Lotus Hotel. During their conversation, one guest accidentally reveals enough information for us to identify one of their online accounts. Our objective is to analyze the conversation, locate the hidden account, and recover the flag.

---

# Objective

- Analyze the provided conversation.
- Extract useful identifying information.
- Pivot using OSINT techniques.
- Locate the hidden account.
- Recover and decode the flag.

---

# Solution

## Step 1 – Analyze the Conversation

The first step was to carefully read the conversation instead of skimming through it. One piece of information immediately stood out:

```text
lambobytelotushotel@gmail.com
```

The challenge tags included **Hashing**, suggesting that the email address itself would play an important role during the investigation.

---

## Step 2 – Pivot Using Gravatar

Gravatar profiles are identified using the **MD5 hash of an email address**.

After generating the MD5 hash for:

```text
lambobytelotushotel@gmail.com
```

I navigated to the corresponding Gravatar profile using the generated hash.

---

## Step 3 – Inspect the Profile

The Gravatar profile belonged to **Lambo** from **Byte Lotus Hotel**.

The profile contained the following message:

```text
Funny thing about email hashes, they follow you places you didn't expect.
Glad you found the right corner of the internet!
Here is your prize:

VENNe1MzY3JIVF9QcjBmaWwzX0g0c19iMzNuX0lkZW50aXR5M2R9
```

---

## Step 4 – Decode the Prize

The provided string was **Base64** encoded.

Decoding it revealed the flag:

```text
TCM{S3crHT_Pr0fil3_H4s_b33n_Ident1ty3d}
```

---

# Flag

```text
TCM{S3crHT_Pr0fil3_H4s_b33n_Ident1ty3d}
```

---

# Key Takeaways

- Always read challenge descriptions carefully; small details often become the starting point for an OSINT investigation.
- Gravatar identifies user profiles using the **MD5 hash** of an email address.
- Hashing and encoding serve different purposes:
  - **MD5** was used to locate the Gravatar profile.
  - **Base64** was used to encode the final flag.
- Effective OSINT often involves chaining together small clues until they reveal the target.

---

**Status:** ✅ Solved

**Technique:** Email Enumeration → MD5 Hash Pivot → Gravatar Enumeration → Base64 Decoding → Flag Recovery