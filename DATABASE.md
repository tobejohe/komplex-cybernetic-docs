# Database Schema (Torus Core)

**Engine**: Cloudflare D1 (SQLite)
**Binding**: `DB` (in `wrangler.toml`)

## Tables Overview

### 1. Identity & Access
- **`users`**: The central entity (Soul ID).
    - `id` (UUID): Internal Soul ID.
    - `nostr_pubkey`: Sovereign 256-bit Hex Identity.
    - `consent_bundle_version`: Required for compliance.
    - `preferences`: **DEPRECATED**. Use `user_memory` (category: `PREFERENCES`).
- **`authorizations`**: Sovereign Capability-Based Access Control (Single Source of Truth for Roles).
    - `capability_mask`: Integer bitmask (BigInt safe).
    - `scope`: Access boundary (`*`, `agency:id`).
    - `label`: Human-readable role name (e.g., 'Master Admin').
- **`identities`**: Link between External IDs and Soul IDs.
    - `provider`: `telegram`, `web`.
    - `provider_id`: Blind-index hash of platform ID.
    - `encrypted_address`: Encrypted raw ID (for proactive contact).
    - `is_active`: 1=active, 0=inactive.
    - `metadata`: JSON blob for lifecycle events (e.g. `unlinked_at`).
- **`auth_tokens`**: Short-lived tokens for session linking and Telegram verification.
- **`sessions`**: Web Session management (24h lifespan).

### 2. Economy & Wallet
- **`wallets`**: Stores `balance` and `total_earned` in **Core Cycles (CC)**.
- **`transactions`**: Ledger for all CC movements.
    - **Refill Logic**: Daily refills are capped by `ECONOMY_REFILL_CAP`. The granted amount is `min(DAILY_REFILL, REFILL_CAP - current_balance)`.

### 3. State & Memory
- **`messages`**: Chat History (The Message Ledger).
- **`audit_logs`**: Administrative Audit Trail (Action, Actor, Target, Changes).
- **`user_memory`**: Long-term facts and specialized preferences (LTM).
    - **Category `PREFERENCES`**: The modern store for user settings (e.g., `debug_mode`, `theme`, `lang`).
    - **Category `CHARACTERISTICS`**: Inferred user traits for personalized AI interaction.
- **`reports`**: Security and Bug reports.
- **`navigation_faults`**: Logs of 404/System errors used for Fraud Prevention and UX auditing.

### 4. Infrastructure & Tenancy
- **`agents`**: Definition of AI Personas. **Note**: Locally stored JSON definitions in `src/agents/*.json` are deprecated. Use the Admin Portal for management.
- **`tools`**: Available functions for the AI. Definitions in `src/tools/definitions/*.json` remain the source of truth for synchronization.
- **`tenants`**: Logical separation of administrative scopes.
- **`communication_channels`**: Dynamic bot management (Multi-Platform / Multi-Instance).
    - `platform`: `telegram`, `discord`, etc.
    - `slug`: Webhook target identifier (unique URL path).
    - `encrypted_credentials`: AES-GCM encrypted JSON blob (Tokens/API Keys).
    - `default_agent_slug`: The agent (slug) that should respond to this channel by default.

### 5. Special Identities
- **`SYSTEM`**: The internal actor for automated treasury actions.
    - **Identity**: `id = 'SYSTEM'`.
    - **Provisioning**: Managed via `scripts/sync_system.js` (`npm run system:push`).

## Migrations

Migrations are stored in `migrations/`.
- **0000_v1_baseline**: Initial schema.
- **0060-0064**: Economy update & Legacy cleanup (removed `pot_u`).
- **0081_authorizations_system**: Professional MBAC (Bitmask-Based Access Control).
- **0090_communication_channels**: Dynamic bot management (Multi-Platform).
- **0091_add_default_agent_slug**: Added agent-to-bot mapping for dynamic routing.

## Backup & Recovery

The system employs a decentralized backup strategy for the hybrid data model:

1.  **GitOps Backups**: Prompts, Tools, and Legal Texts are backed up via standard Git versioning.
2.  **Configuration Snaphots**: The `scripts/backup_db_config.js` tool exports governance-critical tables (`agents`, `prompts`, `tenants`, `tools`, `legal_texts`) to timestamped JSON files in the `backups/` directory.
3.  **D1 Backups**: Full database backups follow Cloudflare D1's internal retention policy.
