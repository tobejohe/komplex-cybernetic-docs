# Internationalization (i18n) System

**Version**: 3.0 (Cross-Project Modular)
**Components**: Torus Core (Worker) & Frontend Terminal (SPA)

## Overview

The i18n system in Komplex [CC] follows a domain-driven, modular architecture. It is designed to be lean, performant, and consistent across the Backend (Cloudflare Worker) and the Frontend (SPA).

## Core Architecture (Worker)

The Worker utilizes a **Facade Pattern** to manage translations without breaking existing code.

### 1. File Structure
Translations are split by language and domain within `src/locales/`:
- `de/`, `en/`, `fr/`, etc.
- **Domains**:
- **Domains**:
    - `telegram.json`: Static bot commands, labels, and fallback messages.
    - **Note**: Structural commands like `/start` are now increasingly handled via AI-native `[SYSTEM_EVENT]` prompts to allow for dynamic, persona-aware responses without static template constraints.
    - `terminal.json`: Terminal gate and auth labels.
    - `legal.json`: Legal texts and consent labels.
    - `ui.json`: Global UI elements.

### 2. Orchestrator (`src/locales/messages.js`)
The orchestrator aggregates all JSON modules. It uses modern ESM static imports with JSON attributes:

```javascript
import de_telegram from './de/telegram.json' with { type: 'json' };
// ... orchestration
export const MESSAGES = { de: { ...de_telegram, ... }, ... };
```

### 3. Fallback Chain
The `getMessage(lang, key)` function implements a robust fail-safe mechanism:
1. **Target Locale**: Search in the requested language (e.g., `fr`).
2. **Global Fallback (EN)**: If not found, use English translation.
3. **Source Fallback (DE)**: If still missing, use German (system origin).
4. **Error Mask**: Returns `[MISSING: key]` if the key is completely undefined.

## Frontend Architecture (Terminal SPA)

The Frontend follows a similar **Pull Pattern**. Views use technical keys resolved by a lightweight orchestrator in the browser.

### Orchestrator Implementation
Templates destructure labels provided by the UI core:
```javascript
export function renderView({ web_terminal_title }) { ... }
```

## Developer Guidelines

### Adding New Labels
1. Identify the domain (telegram, terminal, legal, ui).
2. Add the key-value pair to `src/locales/[lang]/[domain].json`.
3. **Consistency**: Ensure the key exists at least in `de` and `en`.

### Best Practices
- **Atomic Keys**: Keep labels small and reusable.
- **No Logic in JSON**: Translations should not contain complex logic, only simple placeholders if necessary.
- **Manual Sync**: When adding new languages, ensure all essential `telegram` and `legal` keys are present to maintain system functionality.

## Benefits
- **Maintainability**: Smaller files prevent merge conflicts.
- **Performance**: Static imports allow Cloudflare Workers to bundle efficiently.
- **Scalability**: New languages (e.g., Japanese, Chinese) can be added by simply creating a new folder and updating the orchestrator imports.
