---
title: "Playwright Test Automation in Practice: MCP, Codegen, and API Setup"
date: 2026-07-02T12:10:00+09:00
draft: false
type: "post"
categories:
  - Tech Notes
tags:
  - Playwright
  - Test Automation
  - MCP
  - Codegen
  - API Test
  - QA
author: "coherence"
---

## Conclusion First

I tried two different paths for Playwright UI automation:

1. MCP-driven AI execution
2. Playwright Codegen recording

MCP was powerful but too slow for my day-to-day case execution, so I moved to Codegen for UI scenarios. That gave me a much faster path to stable UI tests.

For test data preparation, I realized UI-only setup is inefficient, and API-based setup is the right direction. I am still refining that part, but this post documents a practical setup you can start using today.

---

## Why I Changed My Approach

My original idea was: "Let AI execute test cases for me through MCP."

That worked functionally, but execution speed and iteration time became the bottleneck. For quick trial-and-error on selectors and user flows, I needed something faster.

So I switched to Codegen:

- record actions,
- generate Playwright code in real time,
- and quickly convert that into maintainable test cases.

Result: UI automation moved forward much faster.

---

## Environment

- Node.js 18+
- Playwright (TypeScript)
- VS Code
- Optional MCP client/server integration for AI-driven browser control

---

## 1) MCP Setup for AI-Driven Execution

### What MCP is useful for

MCP is good when you want AI to:

- explore pages,
- run scenario-like steps,
- assist investigation and debugging.

### Basic setup idea

Depending on your MCP client, configuration style differs, but conceptually you register a Playwright-capable MCP server and allow browser actions.

Example (conceptual):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "<playwright-mcp-server-package>"]
    }
  }
}
```

### Why I did not use MCP as the main path

- Great flexibility, but slower feedback loop for repetitive case authoring.
- For frequent small edits (selector fix, assertion tuning), Codegen + direct coding felt much faster.

---

## 2) Codegen Setup for Fast UI Test Authoring

### Project initialization

```bash
npm init playwright@latest
```

Choose:

- TypeScript
- Playwright test directory (for example `tests/`)
- Browsers you need

### Launch Codegen

```bash
npx playwright codegen https://your-app-url.example
```

Useful options:

```bash
# Save authenticated state
npx playwright codegen --save-storage=playwright/.auth/user.json https://your-app-url.example

# Generate script file directly
npx playwright codegen --target=playwright-test --output=tests/ui/generated.spec.ts https://your-app-url.example
```

### Recommended workflow with Codegen

1. Record one clean happy-path flow.
2. Move generated code into page-object or helper functions.
3. Replace brittle selectors with stable locators (`getByRole`, `getByTestId`).
4. Add clear assertions and wait strategy.
5. Re-run in headed and headless mode.

Example command:

```bash
npx playwright test tests/ui --headed
npx playwright test tests/ui
```

---

## 3) API Test Setup for Fast Test Data Preparation

UI setup for preconditions is usually slow. API setup is faster and more deterministic.

### Base API test structure

Create `tests/api/data-setup.spec.ts`:

```ts
import { test, expect, request } from '@playwright/test';

test('prepare test data via API', async () => {
  const api = await request.newContext({
    baseURL: process.env.API_BASE_URL,
    extraHTTPHeaders: {
      Authorization: `Bearer ${process.env.API_TOKEN ?? ''}`,
      'Content-Type': 'application/json'
    }
  });

  const createRes = await api.post('/test-data/entities', {
    data: {
      type: 'sample',
      status: 'active'
    }
  });

  expect(createRes.ok()).toBeTruthy();

  const body = await createRes.json();
  expect(body.id).toBeDefined();
});
```

### Use API setup before UI tests

Pattern A: run API suite first in CI.

```bash
npx playwright test tests/api
npx playwright test tests/ui
```

Pattern B: call API setup in `globalSetup`.

`playwright.config.ts`:

```ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  globalSetup: './tests/setup/global-setup.ts',
  use: {
    baseURL: process.env.UI_BASE_URL
  }
});
```

`tests/setup/global-setup.ts`:

```ts
import { request, type FullConfig } from '@playwright/test';

async function globalSetup(_: FullConfig) {
  const api = await request.newContext({
    baseURL: process.env.API_BASE_URL,
    extraHTTPHeaders: {
      Authorization: `Bearer ${process.env.API_TOKEN ?? ''}`
    }
  });

  const res = await api.post('/test-data/bootstrap', {
    data: { scenario: 'ui-smoke' }
  });

  if (!res.ok()) {
    throw new Error(`Global setup failed: ${res.status()} ${res.statusText()}`);
  }
}

export default globalSetup;
```

### Cleanup strategy (important)

Always add teardown to remove test data. If not, data pollution will break reproducibility.

---

## 4) Copy-Paste Setup Blocks (Directly in This Blog)

Below are the exact files I use as a starting point.

### `.env.example`

```env
# Base URLs
UI_BASE_URL=https://example-ui.local
API_BASE_URL=https://example-api.local

# API auth for setup/teardown (use a non-production token)
API_TOKEN=replace-with-test-token

# Optional controls
TEST_DATA_SCENARIO=ui-smoke
TEST_DATA_CLEANUP_PATH=/test-data/cleanup
```

### `package.json` scripts (`test:ui`, `test:api`, `test:all`)

```json
{
  "name": "playwright-automation",
  "private": true,
  "version": "1.0.0",
  "scripts": {
    "test:ui": "playwright test tests/ui",
    "test:api": "playwright test tests/api",
    "test:all": "npm run test:api && npm run test:ui"
  },
  "devDependencies": {
    "@playwright/test": "^1.55.0",
    "dotenv": "^16.4.5"
  }
}
```

### `playwright.config.ts` (with setup + teardown)

```ts
import { defineConfig } from '@playwright/test';
import * as dotenv from 'dotenv';

dotenv.config();

export default defineConfig({
  testDir: './tests',
  globalSetup: './tests/setup/global-setup.ts',
  globalTeardown: './tests/setup/global-teardown.ts',
  use: {
    baseURL: process.env.UI_BASE_URL,
    trace: 'on-first-retry'
  }
});
```

### `tests/setup/global-teardown.ts` (auto cleanup)

```ts
import { request, type FullConfig } from '@playwright/test';
import * as dotenv from 'dotenv';

dotenv.config();

async function globalTeardown(_: FullConfig) {
  const baseURL = process.env.API_BASE_URL;
  const token = process.env.API_TOKEN;
  const cleanupPath = process.env.TEST_DATA_CLEANUP_PATH || '/test-data/cleanup';
  const scenario = process.env.TEST_DATA_SCENARIO || 'ui-smoke';

  if (!baseURL) {
    console.warn('[teardown] Skip cleanup: API_BASE_URL is not set.');
    return;
  }

  if (!token) {
    console.warn('[teardown] Skip cleanup: API_TOKEN is not set.');
    return;
  }

  const api = await request.newContext({
    baseURL,
    extraHTTPHeaders: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });

  const response = await api.post(cleanupPath, {
    data: { scenario }
  });

  if (!response.ok()) {
    const body = await response.text();
    throw new Error(
      `[teardown] Cleanup failed: ${response.status()} ${response.statusText()} - ${body}`
    );
  }

  await api.dispose();
}

export default globalTeardown;
```

### Command usage

```bash
npm run test:api
npm run test:ui
npm run test:all
```

---

## 5) Suggested Folder Structure

```text
tests/
  ui/
    smoke.spec.ts
    regression.spec.ts
  api/
    data-setup.spec.ts
    health.spec.ts
  setup/
    global-setup.ts
    global-teardown.ts
playwright.config.ts
.env
```

---

## 6) Pros and Cons from My Experience

### MCP

Pros:

- Flexible, exploratory, and helpful for AI-assisted investigation.
- Good for ad-hoc scenario execution.

Cons:

- Too slow for high-frequency test authoring loops.
- Harder to standardize as the primary path for large regression packs.

### Codegen

Pros:

- Very fast for building initial UI scripts.
- Great for selector discovery and flow capture.
- Easy to iterate while watching the browser.

Cons:

- Raw generated code needs refactoring.
- Without cleanup, tests can become brittle.

### API Test for Data Setup

Pros:

- Much faster than preparing data via UI.
- More deterministic and CI-friendly.
- Reduces UI test runtime significantly.

Cons:

- Requires understanding of service contracts and auth.
- Needs robust cleanup to avoid dirty test environments.

---

## 7) My Current Practical Strategy

I now use a hybrid strategy:

1. Use MCP for investigation and AI-assisted exploration.
2. Use Codegen for fast UI script creation.
3. Use API tests to prepare preconditions and test data.
4. Keep UI tests focused on user-visible behavior only.

This gives the best balance between speed and reliability.

---

## Final Takeaway

If your Playwright automation feels slow, separate concerns:

- exploration (MCP),
- UI flow authoring (Codegen),
- data preparation (API test).

For me, this separation was the turning point from "automation is possible" to "automation is practical."

