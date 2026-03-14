# Spartan Node Architecture (SNA)

**Version:** 1.4.0 ("Spartan Edition")  
**Status:** Implemented  
**Core Principles:** Data Sparsity, Sovereign Identity, Passive Discovery.

## 1. The Problem: Identity Inflation

In previous versions, every Nostr persona (Node) required:
1. A separate entry in the `identities` table.
2. A separate entry in `user_memory` for the encrypted private key.
3. Manual state management to keep list of owned nodes in sync.

This led to "Identity Inflation" and increased database complexity as users created dozens of ephemeral nodes.

## 2. The Spartan Solution: Deterministic Derivation

The **Spartan Node Architecture** eliminates persistent storage for Node private keys. Instead, it derives them on-the-fly from the user's **Master Identity**.

### Key Derivation Mechanism
Identity is derived using HMAC-SHA256:
`Node_Entropy = HMAC-SHA256(Master_Private_Key, "node:" + handle)`

- **Master_Private_Key**: The decrypted private key of the user's primary managed identity (`role: primary_human`).
- **Handle**: The unique handle chosen for the node (e.g., "techno_club_berlin").

**Benefits:**
- **Zero Storage**: No private keys are stored in the database for Nodes.
- **Self-Healing**: If the database is lost, all identities can be recovered as long as the Master Key is safe.
- **Portability**: The same Master Key consistently generates the same Node identities across any system implementing the protocol.

## 3. Passive Discovery (P-Tag Linkage)

Since we no longer track nodes in a dedicated table, we use the **Nostr Event Graph** for discovery.

### Genesis Event (Kind 30001)
Every Node is initialized via a `kind:30001` (Replaceable Set) event. To enable discovery, the Spartan Edition enforces a **P-Tag Linkage**:

```json
{
  "kind": 30001,
  "pubkey": "<Node_Pubkey>",
  "tags": [
    ["d", "my_handle"],
    ["p", "<Master_Pubkey>"],
    ["version", "1.4.0"]
  ],
  "content": "{...meta...}"
}
```

- **The `p` tag**: Points to the user's **Master Public Key**.
- **Discovery Flow**: To list all nodes, the system simply queries `DB_EVENTS` for all `kind:30001` events where the `p` tag matches the user's current Master Pubkey.

## 4. Operational State (user_memory)

The system only tracks the **current pointers** in the volatile `user_memory` table to maintain the UI session state:

| Key | Value | Description |
| :--- | :--- | :--- |
| `active_node_pubkey` | `<pubkey>` | Pointer to the currently active identity. |
| `active_node_handle` | `@handle` | Used for deterministic derivation of the active key. |

## 5. Security Implications

1. **Master Key Criticality**: The Master Key is now the single point of failure for all derived identities.
2. **Handle Collisions**: While derivation is deterministic, handle uniqueness is enforced at the database level to prevent two different master keys from claiming the same human-readable handle on the same platform.
3. **Decay**: Nodes still follow the NIP-40 expiration protocol. If a node "decays" and the event is purged from `DB_EVENTS`, it disappears from the list until a new Genesis event is broadcast.
