# GDPR Compliance & Data Deletion Protocol

## Protocol: AMNESIA

The **Amnesia Protocol** is a system-level procedure designed to ensure the irreversible deletion of a user's digital persona while retaining a minimal, anonymized cryptographic proof of their consent history for legal defensibility (Art. 7(1) GDPR).

### The "Ghost Hash" Mechanism

To balance the right to be forgotten (Art. 17 GDPR) with the controller's burden of proof (Art. 7(1) GDPR), we employ a **Keyed-Hash Message Authentication Code (HMAC)** mechanism.

1.  **Input Data**: The user's Nostr Public Key (which they essentially own/control).
2.  **Secret Key**: A server-side environment variable `AMNESIA_SECRET` (never exposed to client).
3.  **Algorithm**: HMAC-SHA256.

```javascript
GhostHash = HMAC(SHA256, Key=AMNESIA_SECRET, Message=UserNostrPuKey)
```

The resulting hash (truncated to 16 hex characters) serves as the **Ghost ID**.

### Data Handling during Deletion

When a user executes `/delete CONFIRM`:

1.  **Ghost ID Generation**: The system calculates the Ghost ID using the user's current public key.
2.  **Consent Log Tombstoning**: The `user_id` in the `consent_log` table is updated to `GHOST_<GhostID>`. The original link to the user table is broken.
3.  **Wallet Anonymization**: The `wallets` entry is deactivated and the `user_id` is updated to `VOID_<GhostID>` to preserve system-wide balance integrity without personal reference.
4.  **Full Deletion**: The user record is DELETED from the `users` table.
    *   **Cascading Deletion**: Identities, Auth Tokens, User Memory, Authorizations.
    *   **Nullification**: Messages, Transactions, Reports are set to NULL user_id.

### 2.5. Psychometric Anonymization (The Observer)

To further reduce data toxicity, we employ the **Observer Engine**:
1.  **Distillation**: Sensitive chat messages are reduced to 6 abstract psychometric vectors (Elitism, Bandwidth, Hedonism, Entropy, Friction, Vector).
2.  **Vector Storage**: These vectors are stored in Cloudflare Vectorize. They are mathematically irreversible; the original text cannot be reconstructed from the 768-dimensional embeddings.
3.  **Source Erasure**: Once distilled (`is_distilled = 1`), the raw message content is subject to the automated deletion cycle (Retention: 8 weeks).
4.  **Sovereign Link**: The vectors are linked to the `session_id`, allowing the system to maintain a personalized "Vibe" for matching without keeping a record of the specific words used to derive it.

### Retrospective Consent Verification

If a former user disputes having given consent, the verification process is as follows:

1.  **Retrieve User's Public Key**: The user must provide their Nostr Public Key (npub/hex).
2.  **Calculate Ghost ID**: An administrator uses the `AMNESIA_SECRET` and the provided Public Key to re-calculate the HMAC.
3.  **Database Lookup**: Search the `consent_log` table for `user_id = 'GHOST_' + CalculatedHash`.

If a record is found, it proves that:
*   The holder of that Private Key interacted with the system.
*   They accepted the conditions stored in that log entry.
*   The record was anonymized at their request (Amnesia).

If a record is *not* found, it proves that:
*   The holder of that Private Key never interacted with the system, because interaction and data-processing are only possible if consent is given.

**Note**: Without the User's Public Key, the `consent_log` entries are mathematically indistinguishable from random noise and cannot be linked back to a natural person by the system operators.

## Data Sovereignty & Audit Transparency

To comply with Article 30 (Records of processing activities) and Article 15 (Right of access):
1.  **Administrative Audit Trail**: Every privilege escalation or core configuration change is logged with a link to the Administrator's Sovereign ID.
2.  **Access Isolation**: Direct database access is restricted; all data mutations occur through authorized API handlers subject to strict capability checks.
3.  **Traceability**: AI interactions are linked to transient Trace IDs, ensuring logical flow monitoring without compromising the integrity of the sovereign soul ID.
