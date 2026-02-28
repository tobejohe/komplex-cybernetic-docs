# Testing Guide

**Version**: 2.0 (Vitest Recovery)
**Framework**: [Vitest](https://vitest.dev/)

## 1. Overview

The project uses a mocked testing environment to ensure high reliability and fast execution without depending on live Cloudflare infrastructure.

## 2. Test Structure

Tests are located in the `tests/` directory:
- `tests/unit/`: Logic-only tests for individual modules (DB, AI, Encryption, Handlers).
- `tests/integration/`: Cross-module tests simulating actual request/response cycles.

## 3. Running Tests

```bash
# Run all tests once
npx vitest run

# Run tests in watch mode
npx vitest

# Run a specific test file
npx vitest run tests/unit/handlers/telegram.test.js
```

## 4. Mocking Strategy

To accommodate Cloudflare Worker specifics (like D1, Queues, and Vectorize) in a pure Node.js environment:
- **Bindings**: All Cloudflare environment variables (`env`) are mocked in `vitest.config.js` or within individual test files.
- **Global Fetch**: Uses the native Node.js `fetch` (v18+) or a mocked version to simulate API calls to Google/Telegram.
- **Markdown Imports**: A custom Vite plugin in `vitest.config.js` allows importing `.md` prompt files as raw strings (standard Cloudflare Worker behavior).

## 5. Adding New Tests

When adding tests for new agents or handlers:
1. Use `vi.mock` to intercept external dependencies (like `ai.js`).
2. Provide a mock `env` object containing the required bindings (e.g., `DB_DYNAMIC`).
3. If testing prompts, ensure the `.md` file is imported correctly via the raw-text plugin.
