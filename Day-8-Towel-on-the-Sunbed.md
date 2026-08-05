# 🏖️ TryHackMe Hacker Holidays 2026 – Day 8: Towel on the Sunbed

## 📌 Challenge Information

- **Room:** Towel on the Sunbed
- **Category:** Web Exploitation
- **Difficulty:** Medium

## 📝 Challenge Overview

The Ponzi wellness rewards application allowed users to claim a **50-point daily reward**, while access to the **Whale Vault** required **150 points**.

The application was vulnerable to a **race condition (TOCTOU – Time-of-Check to Time-of-Use)**. By sending multiple reward claim requests simultaneously, several requests passed the validation before the server updated the user's claim status.

As a result, multiple rewards were credited from a single daily claim, increasing the account balance high enough to unlock the Whale Vault.

---

## 🔍 Skills Practiced

- Web Exploitation
- Business Logic Vulnerabilities
- Race Conditions
- TOCTOU (Time-of-Check to Time-of-Use)
- API Abuse

---

## ⚙️ Exploitation Steps

1. Register a new account.
2. Log in to the dashboard.
3. Capture the `POST /claim` request.
4. Send multiple identical requests simultaneously.
5. Exploit the race condition before the server marks the reward as claimed.
6. Increase the account balance beyond 150 points.
7. Access the Whale Vault.
8. Retrieve the flag.

---

## 🚩 Flag

```text
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

---

## 📚 Key Takeaways

- Business logic vulnerabilities can have severe security impacts even without traditional injection flaws.
- Race conditions occur when multiple requests exploit a timing gap between validation and state updates.
- Critical operations involving balances, rewards, or transactions should be protected using atomic database transactions or proper locking mechanisms to prevent concurrent execution.