# KOMPLEX [CC] Debugging System

The system uses a Cloudflare KV-based logging mechanism to capture internal errors and events that occur during runtime, providing visibility beyond simple console logs.

## The `KOMPLEX_DEBUG` KV Namespace

Errors and debug events are stored in the `KOMPLEX_DEBUG` KV namespace. 

- **Binding Name**: `KOMPLEX_DEBUG`
- **ID**: `65e0dfeb97a947818847d45b7c4a216e`
- **Retention**: Logs currently have an `expirationTtl` of 7 days (604800 seconds).

## Using the `logDebug` Utility

The utility is located at `src/utils/debug.js`.

### 1. Requirements

The utility only logs if the following condition is met in `wrangler.toml`:
```toml
[vars]
DEV_DEBUG = "true"
```

### 2. Implementation

To log an error or event, import and call `logDebug`:

```javascript
import { logDebug } from '../utils/debug.js';

// Default Level: ERROR
await logDebug(env, 'registration', error);

// With Level and Data
await logDebug(env, 'api_call', 'Data received', { payload: data }, 'INFO');
```

### 3. Parameters

- `env`: The Cloudflare environment object.
- `category`: String identifying the log area (e.g., `registration`).
    - **TELEGRAM**: Bot interaction, dispatcher routing, and credential decryption events.
    - **ECONOMY**: Core Cycle transactions and daily refills.
    - **GATEKEEPER**: Security blocks and input sanitization logs.
- `message`: Error object or string message.
- `data` (optional): Object containing additional context.
- `level` (optional): `INFO`, `WARN`, or `ERROR` (default).

### 4. Key Format in KV

Logs are stored with the following key pattern:
`log:[LEVEL]:[category]:[ISO_TIMESTAMP]`

Example: `log:ERROR:registration:2026-02-21T18:45:12.345Z`

### 5. Content Format

The stored value is a JSON string:

```json
{
    "timestamp": "2026-02-21T18:45:12.345Z",
    "level": "ERROR",
    "category": "registration",
    "message": "Specific message",
    "stack": "Stack trace (if error)",
    "context": {
        "environment": "production",
        "some_data": "..."
    }
}
```

## Admin API (Accessing Logs)

Authorized admins can access logs via the Core API:

### List Logs
`GET /admin/debug/list`
Returns a list of keys for all debug logs.

### View Log
`GET /admin/debug/get?key=log:ERROR:registration:...`
Returns the content of a specific log entry.

## 6. Admin Log Explorer
The system provides a unified **Log & Error Explorer** in the Admin Console (`basement.komplex.cc`).
- **Path**: `/logs` (Rights required: `TELEMETRY_VIEW`).
- **Tabs**: 
    - **AI Telemetry**: Live trace monitoring.
    - **Audit Logs**: Administrative action trail.
    - **Debug (KV)**: Visual browser for the `KOMPLEX_DEBUG` KV namespace.
    - **Check ai_telemetry**: Errors are logged with a `trace_id`.
- **Search KV (KOMPLEX_DEBUG)**: Search for the `trace_id` in the `KOMPLEX_DEBUG` KV namespace.
- **Analyze Prompt Registry**: Verify the `gatekeeper` agent and its associated system prompts exist in the `agents` and `prompts` tables.

### Tracing "Security Core Offline" Fatalities
If the bot returns a hard "Security Core Offline" or "Neural Core Misconfigured" error, it means the Orchestration pipeline has crashed.

1. **Locate the Trace ID**: The error response includes the `trace_id`.
2. **Access Debug Logs**: Use the Admin Console or CLI to inspect the `KOMPLEX_DEBUG` KV namespace. 
3. **Filter by Category**: Look for logs categorized as `CONFIG` or `GATEKEEPER`. 
4. **Identify the Root Cause**:
    - **Missing Model Config**: **`AI_CONFIG_ERROR`**: The agent must have an `endpoint` listed in its `model_config`. Silent fallbacks are prohibited.
    - **Invalid JSON**: Verify the `model_config` parsing doesn't crash.
    - **AI Provider Quota**: Look for 429 or 503 errors from the AI adapter.
    - **Decryption Failure**: **`Decryption failed for agent keys`** in KV logs. Verify the `CREDENTIAL_MASTER_KEY` environment variable matches the key used during `encrypt-tool.js` execution.

## 7. Agent Health Scan
The system automatically monitors agent health. Use the **Admin -> Agents** page to see the current status:
- 🟢 **Clean**: Ready for orchestration.
- 🟡 **Operational Errors**: Connected but reporting runtime errors (Check Logs).
- 🔴 **Configuration Error**: Missing endpoint or malformed settings.

## 8. Automated & Characterization Testing

The system uses **Vitest** for unit and integration testing, optimized for the Cloudflare Workers runtime.

### 8.1. Running Tests
```bash
# Run all tests once
npm run test:run

# Run tests in watch mode
npm run test
```

### 8.2. Characterization Strategy
To ensure behavioral parity during refactoring, the system employs **Characterization Tests** (located in `tests/unit/`). 
- **Goal**: Capture the current behavior of a service (even if it's "messy") and freeze it with tests before modularizing the code.
- **Verification**: Refactored code must pass the same characterization suite as the original code.

### 8.3. Resilience Debugging
The system uses the `withTimeout` utility (from `src/utils/db.js`) to prevent hanging turns. If you suspect a database call is timing out:
1. Check `ai_telemetry` for `system_error` events.
2. Monitor `logDebug` for `D1_TIMEOUT` or `DATABASE_ERROR` entries.

> [!TIP]
> If an IP is banned, it will no longer generate fault logs, as all traffic is blocked at the gateway level. To resume debugging for an IP, remove the `ban:[IP]` key from the `RATE_LIMIT` KV namespace.
