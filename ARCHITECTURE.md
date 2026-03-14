# System Architecture (Torus Core)

**Version**: 4.7 ("The Governance Update")
**Runtime**: Cloudflare Workers
**Database**: Cloudflare D1 (SQLite)

## Overview

The Torus Core is a serverless "AI Operating System". It does not run continuously but reacts to events (Webhooks, HTTP Requests, Cron Triggers).

### Core Components

1.  **The Gateway (`src/index.js`) & Producer (`src/handlers/http.js`)**
    - The entry point for all HTTP traffic.
    - Handles **Feature-Toggle Routing**: Distinguishes between legacy endpoints (Auth, Profile, Admin) routed to `src/core/router.js` and pure AI chat (`/api/chat`), routed to the new Producer layer.
    - **Synchronous Membrane**: Executes the strict `Gatekeeper` (via `generateText`) and generates the AI's immediate response (e.g., Sybil) before storing both in the `messages` table.
    - **Queue Trigger**: Offloads heavy, non-blocking tasks (like Vectorization/Distillation) to the `DISTILLATION_QUEUE`.

2.  **The Observer Engine (Async Consumer in `src/handlers/queue.js`)**
    - Triggered by Cloudflare Queues batch delivery.
    - Reads pending messages from D1 (`is_distilled = 0`).
    - Performs deep AI psychometric distillation across **6 Dimensions** (Elitism, Bandwidth, Hedonism, Entropy, Friction, Vector) using Gemini Flash.
    - Generates 768-dimensional Vector embeddings and inserts them into **Cloudflare Vectorize**.
    - For detailed psychometrics, see [OBSERVER_ENGINE.md](OBSERVER_ENGINE.md).17: 
2.  **The Terminal SPA (`src/templates/terminal.js`)**
    - A custom-built, lightweight SPA (Vanilla JS/CSS).
    - **The Gate**: Entry point for initiation/login.
    - **The Nexus**: User dashboard and control center.
    - **The Purgatory**: Dynamic legal consent enforcement.
    - **SSO Orchestration**: Issues unified session cookies (shared with the Admin Console on `basement.komplex.cc`).
    - **Session Lifecycle**: Handles centralized session invalidation via `GET /logout` (The "Sublimation" protocol) across the apex domain.
    - Handles **Sovereign Identity** (Schnorr signatures) locally in the browser.
    - **i18n**: Modular, domain-driven internationalization system for both Worker and Terminal (see [I18N_SYSTEM.md](I18N_SYSTEM.md)).
3.  **The Nexus (Identity & Permissions)**
    - **Centralized User Loading**: Unified logic via `src/utils/user.js` ensures `userState`, `preferences`, and `capability_mask` are loaded consistently for both Web and Telegram visitors.
    - **Capability-Based Security**: Completely replaced legacy role strings with a bitmask-based permission system (`authorizations` table).
    - **Identity Management**: Supports linking multiple external providers (Telegram, etc.) to a single sovereign soul ID.
    - **Navigation Auditing**: Automatically reports "Sackgassen" (404s) to the security protocol via `/api/log/fault` for fraud prevention analysis.
4.  **Spartan Node Architecture (SNA)**
    - Implements deterministic identity derivation to eliminate redundant key storage.
    - Uses P-Tag based discovery for passive persona management.
    - See [SPARTAN_NODE_ARCHITECTURE.md](SPARTAN_NODE_ARCHITECTURE.md) for details.

4.  **The Orchestration Layer (`src/core/orchestration/`)**
    - **Modular Deconstruction**: Replacing the monolithic orchestrator with specialized handlers:
        - `gatekeeperHandler.js`: Intercepts every message for safety and routing (`ALLOW`/`BLOCK`/`SANITIZE`).
        - `contextAssembly.js`: Manages Memory, Vibe (Stavenhagen Protocol), and History windowing.
        - `executionManager.js`: The central logic loop for AI Thinking and Tool Execution.
        - `resultFinalizer.js`: Post-turn cleanup (thinking strip) and ledger persistence.
    - **Hardening**: Standardized use of `withTimeout` for all database interactions to ensure system resilience against Cloudflare D1 latency.

4.  **The Brain (`src/services/aiCore.js` & `src/services/aiHelpers.js`)**
    - Wraps the LLM (Google Gemini 2.0 Flash).
    - **Strict Governance**: Enforces a "No-Fallback" policy. Missing model endpoints trigger a hard `AI_CONFIG_ERROR`.
    - **Modular Logic**: All context-building and payload assembly is extracted to `aiHelpers.js`.
    - **Vibe Context**: Injects `USER_VIBE_MODE` and `CURRENT_SAVED_VIBE` into the system prompt.
    - **Tool Registry**: Dynamic tool resolution via `src/services/toolRegistry.js`.

5.  **The Nervous System (`src/handlers/`)**
    - **Telegram**: Webhook adapter (`src/handlers/telegram.js`). Supports **dynamic instances** with encrypted tokens. Processes Webhook data via `ctx.waitUntil()` to execute Gatekeeper, Sybil response, Database storage, and Queue triggers in the background without blocking the Telegram 200 OK acknowledgment.
    - **Other Platforms**: Framework-ready for Discord, WhatsApp, and Reddit via the `channels.json` configuration abstract.
    - **Web**: Endpoints for the Terminal SPA (`/api/auth`, `/api/chat`).

4.  **The Economy Service (`src/services/economy.js`)**
    - Manages **Core Cycles (CC)**.
    - **System Actor**: Uses a dedicated `SYSTEM` identity to perform automated transactions (e.g. "Welcome Bonus"), ensuring full referential integrity in the ledger.
    - Ensures atomic transactions for tool use and rewards.

## Data Flow

```mermaid
graph TD
    User[User (Browser/App)] -->|1. Local Manifestation| Crypto[PBKDF2 / Schnorr]
    Crypto -->|2. Signed Identity| Gateway[Gateway: src/index.js]
    Gateway -->|Auth/Profile/Admin| LegacyRouter[Legacy Router]
    Gateway -->|Chat (/api/chat)| Producer[HTTP Producer: src/handlers/http.js]
    
    Telegram[Telegram Webhook] -->|Background Task| Producer
    
    Producer -->|3. Security Check| Gatekeeper[Gatekeeper (Gemini)]
    Producer -->|4. Generate Reply| Sybil[Sybil (Gemini)]
    Producer -->|5. Store Message| D1[(D1 Database)]
    Producer -->|6. Trigger Async| Queue[Cloudflare Queue]
    
    Queue -->|Batch Delivery| Observer[Observer Engine: src/handlers/queue.js]
    Observer -->|Read Pending| D1
    Observer -->|Distill Vectors| VectorModel[text-embedding-004]
    Observer -->|Save Embeddings| Vectorize[(Vectorize Index)]
    Observer -->|Mark Done| D1
```

## Security Model

- **Sovereign Identity**: End-to-end cryptographic proof of identity. The server never sees or stores sensitive passwords.
- **Dynamic Consent**: Users must sign specific legal bundles (`REQUIRED_BUNDLE_LEGAL_TEXTS`) before accessing features.
- **Capability-Based RBAC**: The single source of truth for all access control. Access is determined solely by the bitmask.
- **Defensive Banning**: Automated IP-based blocking for clients exceeding navigation fault thresholds.
- **Secure Configuration**: External API keys and bot tokens are stored using **AES-GCM** encryption with **HKDF-SHA256** key derivation.
- **Strict Configuration Policy**: AI agents must have explicit `endpoint` configurations. Fallbacks to default models are strictly prohibited to prevent unmanaged costs and ensure predictable behavior.

## Hybrid Data Strategy

The system utilizes a dual-mode management approach:
-   **Structured GitOps (IDE-Managed)**: 
    - **Prompts & Tools**: Stored as markdown/JS code artifacts for version control and parsed natively using Wrangler's `[Text]` module rules.
    - **Static Configuration**: Hardcoded `src/config/agents.json` and `src/config/channels.json` hold robust operational definitions to bypass D1 database bottlenecks during peak AI routing (e.g. 10,000+ users).
-   **Dynamic Governance (UI-Managed)**: User management, Capability authorizations, and Fault Analytics are managed via the **Admin Portal**.
-   **Backup Protocol**: Governance-critical tables are exported to timestamped JSON snapshots via `scripts/backup_db_config.js`.
