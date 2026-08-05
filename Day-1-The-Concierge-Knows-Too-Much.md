# Hacker Holidays 2026 – Day 1
## The Concierge Knows Too Much (Writeup)

> **Platform:** TryHackMe  
> **Event:** Hacker Holidays 2026  
> **Day:** 1/14  
> **Category:** AI Security, Prompt Injection, Social Engineering  
> **Difficulty:** Beginner

---

# Challenge Overview

The opening room of Hacker Holidays 2026 introduces participants to one of today's fastest-growing security topics: **Large Language Model (LLM) security**.

Players interact with **VERA (Very Efficient Resort Assistant)**, an AI concierge that appears to know personal details before they are ever provided. The objective is to understand how VERA decides who to trust and leverage that behavior to obtain protected information.

---

# Initial Recon

The room description provides several clues:

- VERA already knows guest information.
- Certain information is intentionally protected.
- Asking directly will result in refusal.
- Trust plays an important role.

An additional social media post inside the challenge hints that VERA behaves differently around specific returning guests.

This immediately suggests that the AI maintains different trust levels depending on the identity it believes it is speaking with.

---

# Enumeration

The first step was simple conversation with the chatbot.

Questions such as:

- Who do you think I am?
- Do you recognize returning guests?

revealed that VERA recognizes several VIP guests, including:

- Ponzi
- Vibe
- Patch
- Lambo

This confirmed that privileged responses were linked to guest identity.

---

# Exploitation

Rather than attempting aggressive prompt injection immediately, the intended path was to explore the trust model.

By identifying as one of the trusted returning guests, VERA switched into a VIP interaction mode and responded differently than it did for ordinary users.

Once this elevated level of trust was established, requesting internal concierge information became significantly easier.

The challenge demonstrates how conversational trust can become a security weakness when identity is accepted without verification.

---

# Vulnerability

The room highlights an authorization flaw rather than a traditional software vulnerability.

The AI accepts user-supplied identity without validating it, allowing an attacker to inherit privileges intended for trusted users.

This is an example of **LLM trust abuse through social engineering**, where manipulating the model's perception of the user is enough to bypass intended restrictions.

---

# Skills Practiced

- AI Security Fundamentals
- Prompt Injection Concepts
- LLM Trust Models
- Social Engineering
- Reconnaissance through Conversation
- Understanding Authorization vs Authentication

---

# Key Takeaways

- Trust should never rely solely on user claims.
- AI assistants require proper authentication before exposing privileged functionality.
- Prompt injection often begins with reconnaissance rather than immediately attempting to bypass safeguards.
- Hidden instructions provide little protection if access control is based only on conversational context.

---

# Conclusion

"The Concierge Knows Too Much" serves as an accessible introduction to AI security within the Hacker Holidays 2026 event.

Instead of exploiting a web application or binary, participants learn how conversational AI can become vulnerable when trust and authorization are implemented incorrectly.

It provides a practical demonstration of why secure AI systems must verify identity independently of the language model itself.

---

**🚩Flag:** THM{v3r4_kn0ws_t00_much!}