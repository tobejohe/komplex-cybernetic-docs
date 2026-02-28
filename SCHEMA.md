# Database Schema (torus-db)

**Last Updated:** 2026-02-18  
**Database:** Cloudflare D1 (`torus-db`)  
**ID:** `11d1f497-e6de-408d-b2c2-25c68c2dcd65`

> ⚠️ **Single Source of Truth:** Alle Migrationen werden in `komplex-cybernetic-core/migrations/` verwaltet.

---

## 📋 Table Overview

| Migration              | Tables                                                                                                                                                                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `0000_v1_baseline.sql` | **Baseline 1.0**: users, identities, auth_tokens, agencies, authorizations, agency_members, user_attributions, prompt_library, prompt_versions, tools, agents, messages, user_memory, reports, ai_telemetry, finance_..., reputation_... |

---

## 1. Identity & Auth

### `users`
| Column                 | Type     | Description               |
| ---------------------- | -------- | ------------------------- |
| id                     | TEXT PK  | UUID                      |
| nostr_pubkey           | TEXT     | Nostr Public Key          |
| consent_bundle_version | TEXT     | Active bundle version     |
| consent_date           | DATETIME | Timestamp of last consent |
| created_at             | DATETIME |                           |

### `auth_tokens`
| Column     | Type          | Description    |
| ---------- | ------------- | -------------- |
| token      | TEXT PK       |                |
| user_id    | TEXT FK→users |                |
| expires_at | INTEGER       | Unix timestamp |

### `identities`
| Column            | Type          | Description                   |
| ----------------- | ------------- | ----------------------------- |
| id                | INTEGER PK    | Auto-increment                |
| user_id           | TEXT FK→users |                               |
| provider          | TEXT          | 'telegram', 'web'             |
| provider_id       | TEXT          | Chat ID or Email (hashed)     |
| encrypted_address | TEXT          | Proactive contact (encrypted) |
| encryption_iv     | TEXT          | IV for address                |
| is_active         | INTEGER       | 1=active, 0=inactive          |

---

## 2. Authorization (Capability-Based RBAC)

### `authorizations`
| Column          | Type     | Description                           |
| --------------- | -------- | ------------------------------------- |
| id              | TEXT PK  | UUID for the authorization grant      |
| user_id         | TEXT     | UUID of the user                      |
| capability_mask | INTEGER  | Bitmask of permissions (e.g. 1, 2, 4) |
| scope           | TEXT     | '*' (Global), 'agency:uuid', etc.     |
| label           | TEXT     | Friendly name (e.g. "Superadmin")     |
| created_at      | DATETIME |                                       |

**Capability Definitions (src/utils/permissions.js):**
- **0x01 (1)**: `TELEMETRY_VIEW`
- **0x02 (2)**: `ERROR_READ`
- **0x04 (4)**: `SYSTEM_CONFIG`
- **-1**: `SUPERADMIN` (Full Access)

---

## 3. Brain (Prompts & Tools)

### `prompt_library`
| Column        | Type    | Description            |
| ------------- | ------- | ---------------------- |
| unique_handle | TEXT PK | e.g. 'kernel_security' |
| content       | TEXT    | Prompt content         |
| version       | INTEGER | Current version        |

### `prompt_versions`
| Column        | Type                   | Description        |
| ------------- | ---------------------- | ------------------ |
| id            | INTEGER PK             |                    |
| prompt_handle | TEXT FK→prompt_library |                    |
| content       | TEXT                   | Historical content |
| version       | INTEGER                |                    |

### `tools`
| Column          | Type    | Description |
| --------------- | ------- | ----------- |
| name            | TEXT PK |             |
| description     | TEXT    |             |
| parameters_json | TEXT    | JSON Schema |
| is_dangerous    | BOOLEAN |             |

### `agents`
| Column       | Type        | Description                |
| ------------ | ----------- | -------------------------- |
| id           | TEXT PK     | UUID                       |
| slug         | TEXT UNIQUE | 'sybil', 'torus', 'cortex' |
| name         | TEXT        | Display name               |
| prompt_stack | TEXT        | JSON Array of handles      |
| tool_stack   | TEXT        | JSON Array of tool names   |

---

## 3. Memory & Logging

### `messages`
| Column   | Type       | Description                       |
| -------- | ---------- | --------------------------------- |
| id       | INTEGER PK | Auto-increment                    |
| user_id  | TEXT       |                                   |
| role     | TEXT       | 'user', 'model', 'tool', 'system' |
| content  | TEXT       | Message content                   |
| metadata | TEXT       | JSON (tokens, etc.)               |

### `user_memory`
| Column     | Type       | Description                                            |
| ---------- | ---------- | ------------------------------------------------------ |
| id         | INTEGER PK |                                                        |
| user_id    | TEXT       |                                                        |
| category   | TEXT       | 'fact', 'preference', 'SECURITY', 'CHARACTERISTICS'    |
| fact_key   | TEXT       | e.g. 'lang', 'totp_secret', 'temp_totp_secret'         |
| fact_value | TEXT       | Values are encrypted for 'SECURITY' category (AES-GCM) |
| confidence | INTEGER    | 0-100 (100 = Verified/Final, <100 = Tentative/Temp)    |

**Special Security Notes:**
- **Category SECURITY**: Stored values MUST be encrypted with `ADDRESSES_CRYPT_KEY`.
- **totp_secret**: Verified TOTP secret for 2FA.
- **temp_totp_secret**: Temporary TOTP secret during initialization flow.

### `reports`
| Column   | Type       | Description                       |
| -------- | ---------- | --------------------------------- |
| id       | INTEGER PK |                                   |
| user_id  | TEXT       |                                   |
| category | TEXT       | 'SECURITY_BREACH', 'BUG', 'OTHER' |
| content  | TEXT       |                                   |
| status   | TEXT       | 'OPEN', 'CLOSED'                  |

### `consent_log`
| Column         | Type          | Description               |
| -------------- | ------------- | ------------------------- |
| id             | TEXT PK       | UUID                      |
| user_id        | TEXT FK→users |                           |
| bundle_version | TEXT          | Version e.g. '2026-02-18' |
| documents_json | TEXT          | JSON of accepted docs     |
| signature      | TEXT          | Cryptographic proof       |
| timestamp      | DATETIME      |                           |

### `legal_texts`
| Column     | Type     | Description         |
| ---------- | -------- | ------------------- |
| id         | TEXT PK  | Unique ID (type_ts) |
| type       | TEXT     | agb, privacy, etc.  |
| version    | TEXT     | e.g. '202602191024' |
| language   | TEXT     | de, en, etc.        |
| content    | TEXT     | Markdown content    |
| created_at | DATETIME |                     |

---

## 4. Finance & Economy

### `wallets`
| Column       | Type    | Description         |
| ------------ | ------- | ------------------- |
| user_id      | TEXT PK | UUID                |
| balance      | INTEGER | Core Cycles (CC)    |
| total_earned | INTEGER | Lifetime CC balance |
| updated_at   | INTEGER | Unix timestamp      |

### `transactions`
| Column    | Type       | Description                 |
| --------- | ---------- | --------------------------- |
| id        | INTEGER PK | Auto-increment              |
| user_id   | TEXT       |                             |
| amount    | INTEGER    | Amount in CC                |
| type      | TEXT       | 'DEBIT', 'CREDIT'           |
| reason    | TEXT       | 'Welcome Bonus', 'Tool Use' |
| timestamp | INTEGER    | Unix timestamp              |

### `finance_products`
| Column           | Type    | Description             |
| ---------------- | ------- | ----------------------- |
| key              | TEXT PK | 'PATRONAGE', 'PDF'      |
| name             | TEXT    | Display name            |
| price_eur_cents  | INTEGER | Price in cents          |
| tax_rate         | DECIMAL | VAT rate (default 19.0) |
| stripe_link_dev  | TEXT    | Test mode link          |
| stripe_link_prod | TEXT    | Live mode link          |
| is_active        | BOOLEAN |                         |

### `finance_ledger`
| Column           | Type    | Description                           |
| ---------------- | ------- | ------------------------------------- |
| id               | TEXT PK | UUID                                  |
| user_id          | TEXT    |                                       |
| type             | TEXT    | DEPOSIT, PAYOUT, INVOICE, SERVICE_FEE |
| amount_eur_cents | INTEGER | Positive=Income, Negative=Expense     |
| amount_tax_cents | INTEGER | Tax portion                           |
| status           | TEXT    | PENDING, CLEARED, FAILED, REFUNDED    |
| external_ref     | TEXT    | Stripe reference                      |
| metadata         | JSON    | Additional details                    |

---

## 5. Special Identities & Actors

### `SYSTEM` Actor
The `SYSTEM` user is a non-human identity used for administrative and automated actions (e.g., granting welcome bonuses, daily refills, or processing system fees).

- **Purpose**: Ensures referential integrity in the `transactions` and `messages` tables.
- **Rules**:
    - Has a fixed `id = 'SYSTEM'`.
    - Has a dummy `nostr_pubkey` consisting of 64 zeros.
    - Cannot be deleted or modified by normal users.
    - Its wallet balance typically acts as a "mint" or "treasury" (can be zero or negative depending on accounting logic).
- **Role**: Defined in `user_memory` as `SYSTEM` (category: `SYSTEM`, key: `role`).
- **Creation**: Managed idempotently via `scripts/sync_system.js`. This script is part of the `npm run db:reset` lifecycle and should be run during any new environment setup via `npm run system:push`.

---

## 6. Agencies (Model C - Coopetition)

### `agencies`
| Column            | Type        | Description                          |
| ----------------- | ----------- | ------------------------------------ |
| id                | TEXT PK     | UUID                                 |
| name              | TEXT        | Display name                         |
| slug              | TEXT UNIQUE | URL-safe identifier                  |
| region            | TEXT        | Geographic region                    |
| theme             | TEXT        | Topic/category                       |
| performance_score | REAL        | 0.0 - 1.0                            |
| lead_weight       | REAL        | Calculated from score                |
| max_members       | INTEGER     | Member limit                         |
| status            | TEXT        | starter, verified, golden, suspended |
| contract_ref      | TEXT        | Contract reference                   |

### `backend_users`
| Column        | Type             | Description                   |
| ------------- | ---------------- | ----------------------------- |
| id            | TEXT PK          | UUID                          |
| agency_id     | TEXT FK→agencies | NULL for superadmin           |
| email         | TEXT UNIQUE      | Login email                   |
| password_hash | TEXT             | SHA-256 hash                  |
| role          | TEXT             | 'admin', 'superadministrator' |
| last_login    | TEXT             |                               |
| totp_secret   | TEXT             | TOTP Secret (encrypted)       |
| totp_enabled  | INTEGER          | 0=off, 1=on                   |

### `agency_members`
| Column        | Type                  | Description          |
| ------------- | --------------------- | -------------------- |
| id            | TEXT PK               | UUID                 |
| agency_id     | TEXT FK→agencies      |                      |
| email         | TEXT UNIQUE           | Login email          |
| password_hash | TEXT                  |                      |
| role          | TEXT                  | 'member', 'readonly' |
| created_by    | TEXT FK→agency_admins |                      |

### `user_attributions`
| Column        | Type             | Description                   |
| ------------- | ---------------- | ----------------------------- |
| user_id       | TEXT PK          | Telegram User ID              |
| agency_id     | TEXT FK→agencies | Attributed agency             |
| attributed_at | TEXT             |                               |
| source        | TEXT             | 'referral', 'direct', 'event' |

---

## 🔄 Migration Workflow

```bash
# Neue Migration erstellen
touch migrations/00XX_description.sql

# Lokal testen
npx wrangler d1 execute torus-db --file=migrations/00XX_description.sql --local

# Remote ausführen
npx wrangler d1 execute torus-db --file=migrations/00XX_description.sql --remote
```
