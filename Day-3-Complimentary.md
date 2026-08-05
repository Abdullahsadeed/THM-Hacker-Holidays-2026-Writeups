# TryHackMe Hacker Holidays 2026 – Day 3: Complimentary

**Category:** Cloud (AWS)
**Difficulty:** Easy

## Scenario

The Byte Lotus Wellness application provides a frictionless experience by avoiding user registration altogether. Instead of authenticating users traditionally, it silently provisions temporary AWS credentials using an Amazon Cognito Identity Pool.

The objective is to determine how the application obtains AWS access, assess the permissions granted to guest users, and identify whether those permissions expose data belonging to other guests.

---

## Enumeration

Inspecting the client-side JavaScript revealed that the application used Amazon Cognito to obtain unauthenticated AWS credentials.

Important configuration values included:

- AWS Region
- Cognito Identity Pool ID
- DynamoDB table name

The application initialized guest credentials using `AWS.CognitoIdentityCredentials`, confirming that every visitor receives temporary AWS credentials without authentication.

---

## Obtaining Temporary Credentials

Using the AWS CLI, an unauthenticated Cognito Identity was requested.

```bash
aws cognito-identity get-id \
  --identity-pool-id <IdentityPoolID> \
  --region us-east-1
```

The returned Identity ID was then exchanged for temporary AWS credentials.

```bash
aws cognito-identity get-credentials-for-identity \
  --identity-id <IdentityID> \
  --region us-east-1
```

These temporary credentials were exported as environment variables for subsequent AWS CLI commands.

---

## Permission Enumeration

Attempting to enumerate DynamoDB tables resulted in an AccessDenied error.

```bash
aws dynamodb list-tables --region us-east-1
```

Although table enumeration was blocked, the client-side JavaScript had already disclosed the DynamoDB table name.

This highlights an important lesson:

> Client-side code frequently leaks valuable infrastructure information even when cloud permissions appear restricted.

---

## Exploitation

Instead of listing tables, the known table was queried directly.

```bash
aws dynamodb scan \
  --table-name complimentary-GuestWellnessProfiles \
  --region us-east-1
```

The request returned every guest profile stored in the table.

The dataset included:

- Guest names
- Email addresses
- Phone numbers
- Geographic locations
- Passwords
- Internal notes

One guest's notes contained the challenge flag.

---

## Root Cause

The application relied on unauthenticated Cognito identities while assigning overly permissive IAM permissions.

Although the role prevented table enumeration, it still allowed unrestricted scanning of the DynamoDB table.

This violated the Principle of Least Privilege by granting guest users access to records belonging to every other guest.

---

## Security Impact

An attacker could:

- Enumerate guest profiles
- Retrieve personally identifiable information (PII)
- Obtain stored passwords
- Access guest locations
- Read confidential internal notes

No authentication was required.

---

## Remediation

- Remove direct DynamoDB access from guest identities.
- Implement a backend API to enforce authorization.
- Restrict IAM policies to the minimum required actions.
- Use `GetItem` with identity-based authorization instead of unrestricted `Scan`.
- Never expose infrastructure identifiers unnecessarily within client-side JavaScript.

---

## Key Takeaways

- Client-side JavaScript often exposes cloud infrastructure details.
- Amazon Cognito Identity Pools require carefully scoped IAM policies.
- Blocking enumeration alone does not prevent data exposure.
- Least-privilege IAM design is critical when using temporary AWS credentials.


**🚩Flag:** THM{fr33_app_fr33_d4t4!}