# Security & Observability Guide

## 1. Observability (Trace & Telemetry)

### Trace ID
Every interaction (turn) generates a unique `trace_id` in `src/core/orchestrator.js`. This ID is propagated through the entire orchestration pipeline:
- **Phase Handlers:** `gatekeeperHandler`, `contextAssembly`, `executionManager`, `resultFinalizer`.
- **Sub-services:** `promptEngine`, `history`, `toolRegistry`.
- **Database:** Stored in `ai_telemetry` table.
- **Messages:** Stored in `metadata` JSON of `messages` table.

### Administrative Audit Logging (`audit_logs` table)
Separate from AI Telemetry, all administrative actions in the Admin Console are recorded in `audit_logs`:
- **Trigger**: Mutations in Authorization, Agency settings, or User status.
- **Payload**: Includes `actor_id`, `action` (e.g. `GRANT_AUTHORIZATION`), `target_id`, and `changes` (JSON diff).
- **Metadata**: Captures `ip_address` and `user_agent` for forensic accountability.

### Telemetry & Debugging (`ai_telemetry` & `logDebug`)
The system uses a unified logging approach for both AI events and system diagnostics:
- **`logDebug` (KV)**: Captures detailed system internals (errors, stack traces, security transitions). 
    - **Integration**: Automatically called by `sendAlert`. Every system alert is mirrored to the `KOMPLEX_DEBUG` KV namespace for forensic transparency.
    - **Orchestration**: Instrumentally logged during critical phase failures (e.g., Gatekeeper crashes).
- **`ai_telemetry` (D1)**: High-level AI lifecycle events:
    - `prompt_pull`: System Prompt construction.
    - `tool_hit`: Tool execution (success/error).
    - `security_event`: Gatekeeper blocks (High Severity).
    - `system_error`: Orchestrator crashes.

### Gatekeeper Audit (`messages` table)
From Version 4.1 onwards, **every** user message stored in the database includes a `metadata` JSON field with:
- `gatekeeper_verdict`: "ALLOW", "BLOCK", or "SANITIZE".
- `gatekeeper_latency`: Execution time of the security check.
This allows for silent auditing of allowed messages without flooding the telemetry log.

---

## 2. Security Hardening

### Rate Limiting (Defense in Depth)

#### Core Worker (`komplex.cc`)
- **Limiter:** `RATE_LIMIT` (KV Namespace).
- **Strategy:** Token Bucket (sliding window).
- **Rule:** 100 requests / minute per IP.
- **Action:** Returns `429 Too Many Requests`.
- **Implementation:** `src/index.js` (Server-side enforcement).

#### Admin Console (`basement.komplex.cc`)
- **Limiter:** `RATE_LIMIT_ADMIN` (KV Namespace).
- **Strategy:** Access Control Protection.
- **Rule:** 100 requests / minute per IP.
- **Action:** Returns `429 Too Many Requests`.
- **Implementation:** `_middleware.js` (Admin project).

### 2.2 Defensive Banning (Fraud Prevention)

The system identifies and blocks malicious actors (bots/scanners) based on navigation faults (404s).

- **Limiter:** `RATE_LIMIT` (KV Namespace) with prefix `ban:`.
- **Trigger:** Reaching `SECURITY_FAULT_THRESHOLD` (default: 5) navigation faults within 24 hours.
- **Action**: Issues a global ban for `SECURITY_FAULT_BAN_DURATION` (default: 24h), returning a standardized **JSON response** (`{ error: "BAN_ACTIVE" }`) with a `403 Forbidden` status.
- **Enforcement Layer (IDS/IPS)**:
    - **Core Worker**: Intercepts all API and chat traffic.
    - **Admin Middleware**: Intercepts all administrative access to `basement.komplex.cc`.
    - **Frontend Worker**: Intercepts all static asset and HTML entry points to `komplex.cc`.
- **Implementation**: `src/handlers/fault.js`, `src/core/rateLimit.js`, and `_middleware.js` (Admin).
- **Logging**: Every fault is recorded in the `navigation_faults` table (DB_DYNAMIC) for forensic analysis.

### 2.3 Service Hardening (Resilience)

To prevent cascading failures in the distributed Cloudflare environment:
- **Database Timeouts**: All D1 prepared statements are wrapped in `withTimeout`. This ensures that slow queries do not block the worker thread beyond the allowed execution time.
- **Provider Timeouts**: External calls (e.g. Telegram API, AI API) are protected by a 3-5 second timeout.
- **Fail-Closed Gatekeeper**: If the security core (`gatekeeperHandler`) is unreachable or returns an error, the system defaults to `BLOCK` to prevent unauthorized access during a technical failure.
- **Strict Agent Configuration**: AI Providers must have a valid `endpoint`. The system strictly prohibits fallbacks to prevent silent financial risks.
- **Standardized Error Responses**: All security-related failures return a consistent JSON schema or formatted system message with a unique `errorId` for auditing.

---

## 3. Alerting System

### Dual-Channel Dispatch
Alerts are sent via **Telegram** to administrators. The recipients are resolved dynamically:
1.  **Static Fallback:** `TELEGRAM_ADMIN_IDS` in `wrangler.toml`.
2.  **Dynamic Live:** Administrators in the `authorizations` table with `capability_mask` set to Superadmin/SysConfig.
3.  **Broadcast Channels:** Notified via the `communication_channels` configuration.

### Integrated Debugging (KV Mirroring)
Since Version 4.x, all calls to `sendAlert` automatically trigger a `logDebug` entry. This ensures that even if an administrator misses a Telegram notification, the detailed error context (including stack traces) is safely persisted in the `KOMPLEX_DEBUG` KV namespace.

### Trigger Conditions
| Severity     | Event                               | Source                     |
| :----------- | :---------------------------------- | :------------------------- |
| **CRITICAL** | Neural Core Misconfigured           | `src/core/orchestrator.js` |
| **CRITICAL** | System Crash / Orchestrator Failure | `src/core/orchestrator.js` |
| **HIGH**     | Security Block (Gatekeeper)         | `src/core/orchestrator.js` |
| **WARNING**  | Rate Limit Hit (Debounced)          | `src/index.js`             |

---

## 4. Sovereign Key Derivation (Nostr)

### Derivation Path
The system implements a zero-trust identity manifestation:
1.  **Phase I (Client):** `PBKDF2(PasswordA, salt=PasswordA.trim().toLowerCase().replace(/\s/g, '') + SYSTEM_SALT, iterations=100000)` -> `256-bit Entropy`.
2.  **Phase II (Nostr):** `Schnorr.getPublicKey(entropy)` -> `nostr_pubkey` (32-byte Hex).
3.  **Phase III (Auth):** Every sensitive payload is signed locally using the entropy as a private key.

### Security Implications
- **Zero Knowledge:** The server never sees the password.
- **Collision Resistance:** Using the username as salt prevents global rainbow tables.
- **Auditability:** Every action in D1 remains linked to a cryptographic proof of identity.

### 4.1. Credential Protection (AES-GCM + HKDF)
All API tokens and secrets (e.g. Gemini API Keys, Telegram Bot Tokens) are stored in static JSON configuration files (`src/config/agents.json`, `src/config/channels.json`) instead of the D1 database to ensure high throughput routing. They are encrypted using **AES-GCM (256-bit)** encryption with keys derived via **SHA-256 HKDF**.

- **Master Key**: The `CREDENTIAL_MASTER_KEY` environment variable is the root of trust.
- **Key Derivation**: We use HKDF-SHA256 (empty salt) for domain-separated key derivation (`komplex-encryption-v1`).
- **Isolation**: Each entry (agent or channel) uses a unique IV, ensuring that identical plaintexts result in different ciphertexts.
- **Runtime Decryption**: Decryption occurs just-in-time during request processing in `src/handlers/http.js` and `src/handlers/telegram.js`. Failures are logged to the `KOMPLEX_DEBUG` KV namespace (with a 24h TTL) to assist with troubleshooting master key issues.

#### Static Storage Format
Instead of exposing raw environment variables, the system expects the Base64 ciphertext directly in the configuration format:
- **`agents.json`**: `"encrypted_api_key": "<base64_ciphertext>"`
- **`channels.json`**: `"encrypted_token": "<base64_ciphertext>"`

#### Manual Encryption Tool (`encrypt-tool.js`)
To securely prepare tokens for configuration files, a standalone CLI utility is provided:

```bash
# Usage: node encrypt-tool.js <master_key> <cleartext_token>
node encrypt-tool.js my-secret-master-key "123456:ABC-DEF-GHI"

# Output:
# --- ENCRYPTED TOKEN (BASE64) ---
# eala4HWArySGymym5xHN3kfjlw41Ahe/bmDcuHK3Lk3BgG9tgNiE
```
Copy the resulting Base64 string into the `encrypted_api_key` or `encrypted_token` field of your configuration.

---

## 5. Authorization & Capabilities
The system uses a bitmask-based approach for fine-grained access control.

### Capability Bitmask (`CAP`)
The system uses a 64-bit safe bitmask approach for fine-grained control:

| Bit     | Name             | Category | Description                                                   |
| :------ | :--------------- | :------- | :------------------------------------------------------------ |
| `1<<0`  | `TELEMETRY_VIEW` | System   | View AI Traces and system stats.                              |
| `1<<3`  | `USER_MANAGE`    | Identity | List and analyze user profiles.                               |
| `1<<4`  | `SYSTEM_CONFIG`  | Security | **High Privilege**: Manage system commands and core settings. |
| `1<<10` | `ECONOMY_AUDIT`  | Finance  | View and manage transaction logs.                             |
| `-1`    | `SUPERADMIN`     | God Mode | Full system override (all-bits-set).                          |

### Implementation Details
- **Platform Parity**: The `src/utils/user.js` utility ensures that the user's `capability_mask` is resolved identically for Web, API, and Telegram interactions.
- **Middleware**: Blocks access to the Admin Console for users without the required bits for a specific route.
- **Commands**: All administrative chat commands (e.g., `/debug`, `/mode`) require the `SYSTEM_CONFIG` or `SUPERADMIN` capability.

Detailed bit definitions are maintained in `src/utils/permissions.js`.

---

## 6. Shared Session System (SSO)

The system implements a unified session mechanism across subdomains:
- **Cookie Name:** `${SYSTEM_NAME}_SESSION` (e.g., `KOMPLEXCC_SESSION`).
- **Domain:** `.komplex.cc` (Apex domain sharing between `terminal`, `basement`, and `sybil`).
- **SameSite:** `Lax` (Required for cross-subdomain redirection and centralized logout).
- **Lifespan:** 24 hours (Hardened security baseline).
- **Mechanism:** 
    - The **Core Worker** issues the cookie upon successful cryptographic handshake.
    - The **Admin Console** reads the cookie to verify roles against the `authorizations` table.
    - **Logout (Sublimation):** A centralized `GET /logout` (deeplink) on the Core Worker performs a symmetrical cleanup: it deletes the D1 session, expires the browser cookie, and redirects to `URL_MAIN`.
    - Unauthorized redirection points to the main **Terminal UI** (`komplex.cc`).

---

## 7. Security Roadmap (Future Hardening)

### 7.1. CIDR / Subnet Banning (Cluster-Protection)
To prevent botnets from rotating IPs within the same provider, the system will support blocking entire Subnets (e.g., `1.2.3.0/24`) if multiple IPs from that range exceed the fault threshold within a short window.

### 7.2. Rate-Limit Harmonization
Consolidating IP reputation scores across all workers (Core, Admin, Frontend) into a single "Sovereign Trust Score" to prevent distributed flooding attacks across different subdomains.

### 7.3. AI-Driven Anomaly Detection
Integrating the **Observer Agent** to analyze message patterns.

## 8. Agent Health Monitoring (Traffic Lights)

The **Admin Console** implements a real-time health-check system for AI nodes.

- 🔴 **CRITICAL (RED)**: Missing `endpoint` or invalid model configuration. The node is non-functional.
- 🟡 **WARNING (YELLOW)**: Valid configuration, but the node has reported `ERROR` or `CRITICAL` logs in the last 24 hours.
- 🟢 **OPERATIONAL (GREEN)**: Valid configuration and clean diagnostic history.
