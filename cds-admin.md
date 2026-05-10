# CDS Admin Tool

## Overview

CDS (Component Delivery Service) is GoDaddy's microfrontend platform. Teams publish ESM components to a CDN; consuming apps load them at runtime via `cds-loader`. Today, all lifecycle operations — deploying, configuring, managing access — happen through CLI tools or direct Switchboard edits. There is no UI.

This tool is a web app for CDS component owners. It lives at `cds-admin.int.{env}-gdcorp.tools`, is secured by Jomax SSO, and gives engineers a single place to manage everything about a component.

## What it does

**Component dashboard** — Every component gets a multi-page detail view with deep-linkable URLs per tab. The main page shows at-a-glance health; sub-pages for focused work:
- **Main** (`/components/{name}`): RUM sparklines, error sparklines, plain-text notes textarea (auto-save)
- **Manifest** (`/components/{name}/manifest`): Monaco JSON editor + history/rollback + i18n version hash + Slack webhook config
- **Auth** (`/components/{name}/auth`): Jomax/cert/awsiam allowlist management with link to cds-auth docs
- **RUM** (`/components/{name}/rum`): Full Web Vitals charts (LCP/CLS/INP p75) with deployment event markers (from `metaData.updatedAt`)
- **Logs** (`/components/{name}/logs`): Error/log charts with log-level filter (error/warn/info/all; persisted in localStorage per component using key `cds-admin:logLevel:{name}`), deployment markers, **raw log table** (last 100 entries: timestamp, level, message, consuming app, page URL), **errors-by-shopper breakdown** (top shoppers by error count, from `client.user.id` terms aggregation), and a "Open in Kibana Discover →" deep-link
- **Sync** (`/components/{name}/sync`): Copy manifests across environments (dev/test/prod)

**Component list** — Alphabetically sorted, case-insensitive search. Shows `lastUpdated` (from `metaData.updatedAt`) per component. Each user only sees components they have access to. Super-admins (managed via a Switchboard key, not the app) see everything.

**Analytics** — Pulls live data from two Elasticsearch clusters and renders it with Recharts:
- *Logs cluster*: error rate over time (filterable by log level), request volume
- *RUM cluster*: Web Vitals p75 (LCP, CLS, INP)
- Time range picker (1h / 6h / 24h / 7d)
- "Open in Kibana →" deep-links that pre-filter the dashboard to the selected component

## Access control

- All pages require a valid Jomax SSO session (`connect-gd-auth`)
- Component reads and writes both require the user to be on that component's `cds-auth` allowlist
- Super-admins are listed in `cds-admin.admins.jomax` (edited directly in Switchboard) and bypass all per-component checks

## Auth & infrastructure

- **Switchboard writes**: AWS IAM role via SSO. AWS profile is configured per environment in `gasket.js` (`awsProfile`); a pre-startup script checks the session is active and gives a `aws sso login` hint if not. In prod, Katana injects the IAM role directly.
- **Elasticsearch**: Two separate clusters (logs + RUM), each with its own URL and API key. Keys stay server-side; the UI only hits `/api/analytics/*` proxy routes.
- **Deployment**: Katana (GoDaddy's internal deploy platform) to `dev`, `test`, and `prod` internal domains.

---

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Gasket 7 / Next.js admin UI for GoDaddy's Component Delivery Service that lets engineers view and edit Switchboard manifests, manage access, roll back deployments, sync environments, and monitor component health via Elasticsearch.

**Architecture:** Gasket 7 app (ESM, Pages Router) with Express API routes backed by `@wsb/config-api-client` for Switchboard and `@elastic/elasticsearch` for analytics. All routes protected by `connect-gd-auth` Jomax SSO middleware. UI uses Monaco Editor for JSON editing + Recharts for analytics charts.

**Tech Stack:** Gasket 7, Next.js (Pages Router), Express, `@wsb/config-api-client`, `connect-gd-auth`, `@elastic/elasticsearch`, `@monaco-editor/react`, Recharts, Jest + Supertest

---

## File Structure

```
cds-admin/
├── gasket.js                        # Gasket 7 config: makeGasket + plugins
├── package.json                     # type:module, next/node scripts
├── next.config.js
├── .env.local.example
├── scripts/
│   └── check-aws-sso.js             # Pre-startup AWS SSO check; reads awsProfile from gasket.js
├── clients/
│   ├── switchboard.js               # ConfigClient factory — one instance per (app,env); auth via AWS SDK credential chain
│   ├── elasticsearch.js             # ES client wrappers — getLogsClient() for errors/volume, getRumClient() for web vitals
│   └── slack.js                     # Slack incoming webhook sender
├── plugins/
│   └── switchboard-auth.js          # Sets AWS_PROFILE from gasket.config.awsProfile at prepare
├── middleware/
│   └── require-jomax.js             # Reads req.gdAuth.jwt.jomax; 401 if missing
├── lib/
│   └── access-control.js            # isSuperAdmin, canAccess, assertCanAccess (reads + writes)
├── routes/
│   ├── auth.js                      # GET /auth/me, POST /auth/logout
│   ├── config.js                    # GET /api/config — public non-secret config (kibanaDashboards per env)
│   ├── components.js                # registry.{name}: list (filtered), get, put, create, history, rollback, sync
│   ├── i18n.js                      # registry.{name}.i18nVersion: get, put, history, rollback
│   ├── rum.js                       # registry.{name}.rum: get, put, history, rollback
│   ├── notes.js                     # registry.{name}.notes: get, put (plain text, no history)
│   ├── access.js                    # cds-auth {name}: get, put jomax array
│   ├── alerts.js                    # cds-admin config.{name}: get, put, test
│   └── analytics.js                 # ES proxy: errors, volume, rum vitals, raw logs, errors-by-user
├── pages/
│   ├── _app.js                      # Auth gate — redirects to SSO if no jomax session
│   ├── index.js                     # Component list (filtered to user's access; shows lastUpdated)
│   ├── components/
│   │   ├── new.js                   # Create component form
│   │   └── [name]/
│   │       ├── index.js             # Main: RUM sparklines + error sparklines + notes textarea
│   │       ├── manifest.js          # Manifest editor + history/rollback + i18n + Slack alerts
│   │       ├── auth.js              # cds-auth allowlist (jomax/cert/awsiam) + docs link
│   │       ├── rum.js               # Full RUM charts with deployment event markers
│   │       ├── logs.js              # Error/log charts + localStorage log-level filter + markers
│   │       └── sync.js              # Environment sync panel (dev/test/prod copy)
│   ├── docs/
│   │   ├── cds-auth.js              # Static: explains jomax/cert/awsiam auth types
│   │   └── elasticsearch.js         # Static: explains logs + RUM clusters, fields, example docs
│   └── analytics.js                 # Full analytics page (time ranges, deeper dive)
├── components/
│   ├── Layout.js                    # Shell: nav, env selector, user display
│   ├── ComponentNav.js              # Tab bar shared across all /components/[name]/* pages
│   ├── JsonEditor.js                # Monaco wrapper; emits onChange + onValidate
│   ├── Notes.js                     # Plain-text notes textarea with debounced auto-save
│   ├── HistoryTimeline.js           # List of history entries + Rollback button
│   ├── SyncPanel.js                 # Env status badges + Copy env→env buttons
│   ├── AccessList.js                # jomax/cert/awsiam allowlist editor; last-jomax-user protected
│   └── charts/
│       ├── ErrorsOverTime.js        # Recharts LineChart for error count
│       ├── RequestVolume.js         # Recharts LineChart for request count
│       ├── WebVitals.js             # Recharts LineChart for p75 LCP/CLS/INP
│       └── MiniChart.js             # Compact single-metric sparkline for main page
└── test/
    ├── clients/
    │   ├── switchboard.test.js
    │   └── elasticsearch.test.js
    ├── middleware/
    │   └── require-jomax.test.js
    ├── lib/
    │   └── access-control.test.js
    └── routes/
        ├── components.test.js
        ├── i18n.test.js
        ├── rum.test.js
        ├── notes.test.js
        ├── access.test.js
        ├── alerts.test.js
        └── analytics.test.js       # includes level filter tests
```

---

## Data Model (Switchboard apps)

### `cds` app — one key per piece of data per component
| Key pattern | Value | Description |
|-------------|-------|-------------|
| `registry.{name}` | JSON | Manifest: dependencies, engine, js, metaData |
| `registry.{name}.i18nVersion` | string | i18n hash (e.g. `607a011`) |
| `registry.{name}.rum` | JSON | RUM poller config |
| `registry.{name}.notes` | string | Engineer notes, plain text, free-form |

### `cds-auth` app — per-component allowlist
| Key | Value |
|-----|-------|
| `{name}` (e.g. `componentA`) | `{ jomax: [...], cert: [...], awsiam: [...] }` |

### `cds-admin` app — admin tool config (new, user provisions)
| Key | Value | Description |
|-----|-------|-------------|
| `config.{name}` | `{ slackWebhookUrl: string \| null }` | Per-component Slack config |
| `admins` | `{ jomax: ["jroig", "tbogart", ...] }` | **Super-admins** — read/write ANY component. Edited directly in Switchboard (not via the app). |

## Elasticsearch Data Sources

The app reads from two separate Elasticsearch clusters. Both are **read-only** from the app's perspective — the clusters are written by `cds-loader` and the RUM poller, not by this admin tool. API keys stay server-side; the browser only hits `/api/analytics/*` proxy routes.

### Logs cluster — `.ds-cds-api-prod-logs-*`

**Connection:** `ELASTICSEARCH_LOGS_URL` + `ELASTICSEARCH_LOGS_API_KEY`

Captures every CDS API call and error. Written by `cds-loader` when components are loaded on consumer pages.

**Key fields used in this app:**

| Field | Type | Description |
|-------|------|-------------|
| `@timestamp` | date | When the event occurred |
| `event.type` | keyword | Component name (e.g. `bamMediaManagerEsm`) — **primary filter for all log queries** |
| `log.level` | keyword | `error`, `warn`, `info` |
| `message` | text | Log message body |
| `error.stack_trace` | text | Stack trace (present on error-level entries only) |
| `service.name` | keyword | Consuming app that loaded the component (e.g. `websites-godaddy`) |
| `http.request.referrer` | keyword | Page URL where the component was loaded |
| `client.user.id` | keyword | Shopper ID — used for errors-by-shopper aggregation |

**Queries that use this cluster:**
- `queryErrors(componentName, from, to, level)` — `date_histogram` agg filtered by `event.type` + optional `log.level`
- `queryRequestVolume(componentName, from, to)` — `date_histogram` agg on info-level entries matching `componentTypes=` in the message
- `queryRawLogs(componentName, from, to, level, size)` — top-N hits sorted `@timestamp desc`, source-filtered to the 7 fields above
- `queryErrorsByUser(componentName, from, to)` — `terms` agg on `client.user.id`, top 10 by count

**Example document:**
```json
{
  "@timestamp": "2026-01-15T14:23:11.042Z",
  "event": { "type": "bamMediaManagerEsm" },
  "log": { "level": "error" },
  "message": "Failed to load module: network timeout after 3000ms",
  "error": { "stack_trace": "Error: network timeout\n  at fetch (/cds-loader/src/loader.js:142:11)" },
  "service": { "name": "websites-godaddy" },
  "http": { "request": { "referrer": "https://www.godaddy.com/domains/domain-name-search" } },
  "client": { "user": { "id": "shopper-82a4f1c3" } }
}
```

### RUM cluster — `sns-traffic-c1-prod*`

**Connection:** `ELASTICSEARCH_RUM_URL` + `ELASTICSEARCH_RUM_API_KEY`

Captures Core Web Vitals measurements from real users. Written by the RUM poller that runs alongside each component on the page.

**Key fields used in this app:**

| Field | Type | Description |
|-------|------|-------------|
| `@timestamp` | date | When the measurement was taken |
| `customProps.media_library.componentType` | keyword | Component name — **primary filter** |
| `lcp` | float | Largest Contentful Paint (ms) — lower is better, target < 2500ms |
| `cls` | float | Cumulative Layout Shift (unitless) — lower is better, target < 0.1 |
| `inp` | float | Interaction to Next Paint (ms) — lower is better, target < 200ms |

**Queries that use this cluster:**
- `queryWebVitals(componentName, from, to)` — `date_histogram` agg with nested `percentiles` sub-aggs for LCP/CLS/INP p75

**Example document:**
```json
{
  "@timestamp": "2026-01-15T14:23:15.890Z",
  "customProps": {
    "media_library": {
      "componentType": "bamMediaManagerEsm"
    }
  },
  "lcp": 1842.5,
  "cls": 0.04,
  "inp": 112.0
}
```

### Kibana access

Both clusters have Kibana instances. The `kibanaDiscover` config in `gasket.js` per environment provides:
- `logsUrl` — base URL for the logs cluster's Kibana (different between prod and non-prod)
- `logsDataViewId` — the data view ID to pre-select in Kibana Discover

The `buildKibanaDiscoverUrl(kibanaDiscover, componentName, from, to)` helper on the Logs sub-page builds a full Rison-encoded Discover deep-link that pre-filters to the component and time range.

---

## Access Control Model

- **Super admins** (`cds-admin.admins.jomax`) bypass all per-component checks — they can read/write anything in any env.
- **Regular users** see only components where their Jomax username appears in `cds-auth.{name}.jomax` for the current env.
- Writes always require allowlist membership (or super-admin).
- Reads now ALSO require allowlist membership (or super-admin) — fixes a prior gap where any Jomax user could read any component.

**Route pattern:** Every route touching a specific component (GET, PUT, POST, DELETE) must call `assertCanAccess(name, req.user.accountName, env)` before the Switchboard operation. The only exception is `GET /api/components` — which uses `listAccessibleComponents()` to filter instead. This pattern must be applied in every route file in Tasks 4, 6, 7, 9, 10, 11, 12, and the new Notes task — not just writes.

---

## Task 1: Project Scaffold

**Files:**
- Create: `package.json`
- Create: `gasket.js`
- Create: `next.config.js`
- Create: `.env.local.example`

- [ ] **Step 1: Scaffold with create-gasket-app**

```bash
npx create-gasket-app@latest cds-admin --preset @gasket/preset-nextjs
cd cds-admin
```

- [ ] **Step 2: Install dependencies**

```bash
npm install @wsb/config-api-client @wsb/core-config-rules connect-gd-auth \
  @elastic/elasticsearch @monaco-editor/react recharts \
  @ux/button @ux/text @ux/text-entry @ux/spinner @ux/tag

npm install --save-dev jest @jest/globals supertest jest-environment-node \
  @testing-library/react @testing-library/jest-dom
```

- [ ] **Step 3: Replace gasket.js**

```js
// gasket.js
import { makeGasket } from '@gasket/core';
import pluginNextjs from '@gasket/plugin-nextjs';
import pluginExpress from '@gasket/plugin-express';
import pluginLogger from '@gasket/plugin-logger';
import switchboardAuth from './plugins/switchboard-auth.js';

export default makeGasket({
  plugins: [pluginNextjs, pluginExpress, pluginLogger, switchboardAuth],
  express: { routes: './routes/**/*.js' },
  environments: {
    local: {
      hostname: 'localhost',
      awsProfile: 'cds-admin-dev-sso',
      auth: { host: 'sso.dev-godaddy.com', ssoBase: 'https://sso.dev-gdcorp.tools' },
      kibanaDashboards: {
        logs: 'https://<logs-cluster-dev>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<logs-dashboard-id>',
        rum:  'https://<rum-cluster-dev>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<rum-dashboard-id>'
      },
      kibanaDiscover: {
        logsUrl: 'https://usiessp-non-prod-usw2.kb.us-west-2.aws.found.io:9243',
        logsDataViewId: 'b7078ea2-8474-4e6e-b55f-a1a51d1562a0'
      }
    },
    dev: {
      hostname: 'cds-admin.int.dev-gdcorp.tools',
      awsProfile: 'cds-admin-dev-sso',
      auth: { host: 'sso.dev-godaddy.com', ssoBase: 'https://sso.dev-gdcorp.tools' },
      kibanaDashboards: {
        logs: 'https://<logs-cluster-dev>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<logs-dashboard-id>',
        rum:  'https://<rum-cluster-dev>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<rum-dashboard-id>'
      },
      kibanaDiscover: {
        logsUrl: 'https://usiessp-non-prod-usw2.kb.us-west-2.aws.found.io:9243',
        logsDataViewId: 'b7078ea2-8474-4e6e-b55f-a1a51d1562a0'
      }
    },
    test: {
      hostname: 'cds-admin.int.test-gdcorp.tools',
      awsProfile: 'cds-admin-test-sso',
      auth: { host: 'sso.test-godaddy.com', ssoBase: 'https://sso.test-gdcorp.tools' },
      kibanaDashboards: {
        logs: 'https://<logs-cluster-test>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<logs-dashboard-id>',
        rum:  'https://<rum-cluster-test>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<rum-dashboard-id>'
      },
      kibanaDiscover: {
        logsUrl: 'https://usiessp-non-prod-usw2.kb.us-west-2.aws.found.io:9243',
        logsDataViewId: 'b7078ea2-8474-4e6e-b55f-a1a51d1562a0'
      }
    },
    prod: {
      hostname: 'cds-admin.int.gdcorp.tools',
      awsProfile: null,  // Katana injects IAM role directly — no profile needed
      auth: { host: 'sso.godaddy.com', ssoBase: 'https://sso.gdcorp.tools' },
      kibanaDashboards: {
        logs: 'https://<logs-cluster-prod>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<logs-dashboard-id>',
        rum:  'https://<rum-cluster-prod>.kb.us-west-2.aws.found.io:9243/app/dashboards#/view/<rum-dashboard-id>'
      },
      kibanaDiscover: {
        logsUrl: 'https://usiessp-prod-usw2.kb.us-west-2.aws.found.io:9243',
        logsDataViewId: 'e43871e5-f314-4ca9-8eb9-697e1799095c'
      }
    }
  }
});
```

- [ ] **Step 4: Update package.json scripts**

```json
{
  "type": "module",
  "scripts": {
    "dev":   "next dev",
    "build": "next build",
    "start": "node server.js",
    "local": "GASKET_ENV=local node scripts/check-aws-sso.js && GASKET_ENV=local next dev",
    "test":  "node --experimental-vm-modules node_modules/.bin/jest"
  }
}
```

- [ ] **Step 5: Create .env.local.example**

Copy to `.env.local` (git-ignored) and fill in the ES API keys before running locally. For dev/test/prod deployments, these same env vars are injected by Katana secrets — no `.env.local` file is used there.

```bash
# .env.local.example  ← copy to .env.local and fill in values; never commit .env.local
GASKET_ENV=local

# ── Elasticsearch API keys ─────────────────────────────────────────────────────
# LOCAL:       set here in .env.local (get keys from the Elastic Cloud console)
# DEV/TEST/PROD: injected automatically by Katana secrets — no file needed

# Logs cluster — errors + request volume (.ds-cds-api-prod-logs-*)
ELASTICSEARCH_LOGS_URL=https://<logs-cluster>.es.us-west-2.aws.found.io:9243
ELASTICSEARCH_LOGS_API_KEY=

# RUM cluster — web vitals (sns-traffic-c1-prod*)
ELASTICSEARCH_RUM_URL=https://<rum-cluster>.es.us-west-2.aws.found.io:9243
ELASTICSEARCH_RUM_API_KEY=

# ── Other ──────────────────────────────────────────────────────────────────────
SESSION_SECRET=dev-secret-change-in-prod
# SSO host is configured per-environment in gasket.js environments block — no env var needed
# AWS profile is configured per-environment in gasket.js (awsProfile field)
# Run `aws sso login --profile cds-admin-dev-sso` before starting locally
# Kibana dashboard base URLs are configured per-environment in gasket.js (kibanaDashboards.logs / .rum)
# Fill in the placeholder cluster names and dashboard IDs in gasket.js before running
```

- [ ] **Step 6: Create scripts/check-aws-sso.js**

Runs before the dev server. Imports `gasketConfig` to get the `awsProfile` for the current `GASKET_ENV`, then verifies the session is active via `aws sts get-caller-identity`. If not, exits with a clear, actionable error.

```js
// scripts/check-aws-sso.js
import { execFileSync } from 'child_process';
import { gasketConfig } from '../gasket.js';

const env = process.env.GASKET_ENV || 'local';
const profile = gasketConfig.environments?.[env]?.awsProfile;

if (!profile) {
  // Katana/CI injects credentials directly — no profile needed
  process.exit(0);
}

try {
  execFileSync('aws', ['sts', 'get-caller-identity', '--profile', profile], {
    stdio: 'pipe',
    encoding: 'utf-8',
    timeout: 5000
  });
} catch (err) {
  const notFound = err.code === 'ENOENT';
  if (notFound) {
    console.error('❌ ERROR: AWS CLI not found. Install it: https://aws.amazon.com/cli/');
  } else {
    console.error(`
❌ ERROR: AWS SSO session is not active or expired!

This will cause Switchboard authentication to fail with:
   "Token Management Error: Unable to obtain token from SSO"

To fix this, run:
   aws sso login --profile ${profile}

Then restart the dev server.
`);
  }
  process.exit(1);
}
```

- [ ] **Step 7: Create jest.config.js** (required for ESM tests)

```js
// jest.config.js
export default {
  testEnvironment: 'node',
  transform: {},               // disable Babel — we're already ESM
  moduleFileExtensions: ['js', 'jsx', 'mjs'],
  testMatch: ['**/test/**/*.test.js']
};
```

- [ ] **Step 8: Verify dev server starts**

```bash
cp .env.local.example .env.local
# fill in ES keys, then:
aws sso login --profile cds-admin-dev-sso
npm run local
```

Expected: AWS SSO check passes, Next.js compiles, server starts on port 3000.

- [ ] **Step 9: Commit**

```bash
git init && git add .
git commit -m "feat: initial Gasket 7 scaffold"
```

---

## Task 2: Switchboard Client Factory

**Files:**
- Create: `clients/switchboard.js`
- Create: `test/clients/switchboard.test.js`
- Create: `plugins/switchboard-auth.js`

- [ ] **Step 1: Write the failing test**

```js
// test/clients/switchboard.test.js
// Auth is handled entirely by the AWS SDK credential chain (AWS_PROFILE set by
// switchboard-auth.js plugin). The client itself needs no auth options.
import { jest } from '@jest/globals';

const ConfigClientMock = jest.fn().mockImplementation(() => ({
  get: jest.fn(),
  put: jest.fn(),
  getHistory: jest.fn()
}));

jest.unstable_mockModule('@wsb/config-api-client', () => ({ default: ConfigClientMock }));
jest.unstable_mockModule('@wsb/core-config-rules', () => ({ default: {} }));

const { getCdsClient, getCdsAuthClient, getCdsAdminClient } = await import('../../clients/switchboard.js');

test('getCdsClient returns same instance for same env', () => {
  const a = getCdsClient('dev');
  const b = getCdsClient('dev');
  expect(a).toBe(b);
});

test('getCdsClient and getCdsAuthClient return different instances', () => {
  const cds = getCdsClient('dev');
  const auth = getCdsAuthClient('dev');
  expect(cds).not.toBe(auth);
});

test('getCdsAuthClient uses cds-auth app', () => {
  getCdsAuthClient('dev');
  expect(ConfigClientMock).toHaveBeenCalledWith(expect.objectContaining({ app: 'cds-auth' }));
});

test('getCdsAdminClient uses cds-admin app', () => {
  getCdsAdminClient('dev');
  expect(ConfigClientMock).toHaveBeenCalledWith(expect.objectContaining({ app: 'cds-admin' }));
});

test('local env maps to dev switchboard env', () => {
  getCdsClient('local');
  expect(ConfigClientMock).toHaveBeenCalledWith(expect.objectContaining({ env: 'dev' }));
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/clients/switchboard.test.js
```

Expected: `Cannot find module '../../clients/switchboard.js'`

- [ ] **Step 3: Implement clients/switchboard.js**

```js
// clients/switchboard.js
// Auth is entirely handled by the AWS SDK credential chain. AWS_PROFILE is set
// at startup by plugins/switchboard-auth.js from gasket.config.awsProfile.
// No auth options needed here — ConfigClient picks up credentials automatically.
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const ConfigClient = require('@wsb/config-api-client');
const configRules = require('@wsb/core-config-rules');

const cache = {};

function getClient(app, env) {
  const key = `${app}:${env}`;
  if (!cache[key]) {
    cache[key] = new ConfigClient({
      app,
      env: env === 'local' ? 'dev' : env,
      plugins: [configRules],
      cache: { path: null, tts: 30000 }
    });
  }
  return cache[key];
}

export const getCdsClient      = env => getClient('cds', env);
export const getCdsAuthClient  = env => getClient('cds-auth', env);
export const getCdsAdminClient = env => getClient('cds-admin', env);
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npm test -- test/clients/switchboard.test.js
```

- [ ] **Step 5: Create plugins/switchboard-auth.js** (no test needed — pure config injection)

```js
// plugins/switchboard-auth.js
// Sets AWS_PROFILE from gasket.config.awsProfile so the AWS SDK credential chain
// picks up the right SSO profile for the current environment.
// In prod, Katana injects IAM role credentials directly — awsProfile is null there.
export default {
  name: 'cds-admin-switchboard-auth',
  hooks: {
    prepare(gasket) {
      const profile = gasket.config.awsProfile;
      if (profile) {
        process.env.AWS_PROFILE = profile;
        gasket.logger.info(`[switchboard-auth] AWS_PROFILE set to ${profile}`);
      }
    }
  }
};
```

- [ ] **Step 6: Commit**

```bash
git add clients/switchboard.js plugins/switchboard-auth.js test/clients/switchboard.test.js
git commit -m "feat: switchboard client factory with IAM role auth"
```

---

## Task 3: Auth Middleware + Auth Route + Config Route + Error Handler + CSRF

**Files:**
- Create: `middleware/require-jomax.js`
- Create: `middleware/require-same-origin.js` (CSRF defense)
- Create: `middleware/error-handler.js`
- Create: `lib/access-control.js`
- Create: `routes/auth.js`
- Create: `routes/config.js`
- Create: `test/middleware/require-jomax.test.js`
- Create: `test/middleware/require-same-origin.test.js`
- Create: `test/lib/access-control.test.js`
- Create: `test/routes/config.test.js`

- [ ] **Step 1: Write failing tests**

```js
// test/middleware/require-jomax.test.js
import { jest } from '@jest/globals';
import { requireJomax } from '../../middleware/require-jomax.js';

function makeReq(jomaxUser) {
  return {
    gdAuth: jomaxUser
      ? { jwt: { jomax: { getAccountName: () => jomaxUser } } }
      : { jwt: {} }
  };
}

test('calls next() when jomax token present', () => {
  const req = makeReq('jroig');
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };
  const next = jest.fn();
  requireJomax(req, res, next);
  expect(next).toHaveBeenCalled();
  expect(req.user).toEqual({ accountName: 'jroig' });
});

test('returns 401 when no jomax token', () => {
  const req = makeReq(null);
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };
  const next = jest.fn();
  requireJomax(req, res, next);
  expect(res.status).toHaveBeenCalledWith(401);
  expect(next).not.toHaveBeenCalled();
});
```

```js
// test/lib/access-control.test.js
import { jest } from '@jest/globals';

const mockCdsAuthGet = jest.fn();
const mockCdsAdminGet = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsAuthClient: jest.fn(() => ({ get: mockCdsAuthGet })),
  getCdsAdminClient: jest.fn(() => ({ get: mockCdsAdminGet }))
}));

const { isSuperAdmin, canAccess, assertCanAccess, listAccessibleComponents } =
  await import('../../lib/access-control.js');

beforeEach(() => {
  mockCdsAuthGet.mockReset();
  mockCdsAdminGet.mockReset();
});

test('isSuperAdmin true when user is in cds-admin.admins.jomax', async () => {
  mockCdsAdminGet.mockResolvedValue({ jomax: ['jroig', 'tbogart'] });
  expect(await isSuperAdmin('jroig', 'dev')).toBe(true);
  expect(await isSuperAdmin('bob', 'dev')).toBe(false);
});

test('canAccess true when user is on allowlist for component', async () => {
  mockCdsAdminGet.mockResolvedValue({ jomax: [] });
  mockCdsAuthGet.mockResolvedValue({ jomax: ['alice'] });
  expect(await canAccess('componentA', 'alice', 'dev')).toBe(true);
  expect(await canAccess('componentA', 'bob', 'dev')).toBe(false);
});

test('canAccess true for super admin even when not on component allowlist', async () => {
  mockCdsAdminGet.mockResolvedValue({ jomax: ['jroig'] });
  mockCdsAuthGet.mockResolvedValue({ jomax: ['alice'] });
  expect(await canAccess('componentA', 'jroig', 'dev')).toBe(true);
});

test('assertCanAccess throws 403 when denied', async () => {
  mockCdsAdminGet.mockResolvedValue({ jomax: [] });
  mockCdsAuthGet.mockResolvedValue({ jomax: ['alice'] });
  await expect(assertCanAccess('componentA', 'bob', 'dev'))
    .rejects.toMatchObject({ status: 403 });
});

test('listAccessibleComponents returns ALL for super admin', async () => {
  mockCdsAdminGet.mockResolvedValue({ jomax: ['jroig'] });
  // Even if cds-auth has no entries, super admin gets everything
  expect(await listAccessibleComponents('jroig', 'dev', ['a', 'b', 'c']))
    .toEqual(['a', 'b', 'c']);
});

test('listAccessibleComponents filters to user allowlist', async () => {
  mockCdsAdminGet.mockResolvedValue({ jomax: [] });
  // One call to cds-auth.get() returns the full map
  mockCdsAuthGet.mockResolvedValue({
    a: { jomax: ['alice', 'bob'] },
    b: { jomax: ['alice'] },
    c: { jomax: ['charlie'] }
  });
  expect(await listAccessibleComponents('alice', 'dev', ['a', 'b', 'c']))
    .toEqual(['a', 'b']);
});
```

```js
// test/middleware/require-same-origin.test.js
import { jest } from '@jest/globals';
import { requireSameOrigin } from '../../middleware/require-same-origin.js';

process.env.HOSTNAME = 'cds-admin.int.test-gdcorp.tools';

function mockReq(method, origin) {
  return { method, headers: { origin } };
}

test('allows GET without Origin check', () => {
  const next = jest.fn();
  requireSameOrigin(mockReq('GET'), { status: jest.fn() }, next);
  expect(next).toHaveBeenCalled();
});

test('allows POST with matching Origin', () => {
  const req = mockReq('POST', 'https://cds-admin.int.test-gdcorp.tools');
  const next = jest.fn();
  requireSameOrigin(req, { status: jest.fn() }, next);
  expect(next).toHaveBeenCalled();
});

test('rejects POST with mismatched Origin', () => {
  const req = mockReq('POST', 'https://evil.example.com');
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };
  const next = jest.fn();
  requireSameOrigin(req, res, next);
  expect(res.status).toHaveBeenCalledWith(403);
  expect(next).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
npm test -- test/middleware/require-jomax.test.js test/lib/allowlist.test.js
```

- [ ] **Step 3: Implement middleware/require-jomax.js**

```js
// middleware/require-jomax.js
export function requireJomax(req, res, next) {
  const accountName = req.gdAuth?.jwt?.jomax?.getAccountName();
  if (!accountName) {
    return res.status(401).json({ error: 'Jomax authentication required' });
  }
  req.user = { accountName };
  next();
}
```

- [ ] **Step 4: Implement lib/access-control.js**

```js
// lib/access-control.js
import { getCdsAuthClient, getCdsAdminClient } from '../clients/switchboard.js';

// Super admins are listed in cds-admin.admins.jomax — they bypass all per-component checks.
// This key is edited directly in Switchboard (not via the app).
export async function isSuperAdmin(accountName, env) {
  const admins = await getCdsAdminClient(env).get('admins');
  return admins?.jomax?.includes(accountName) ?? false;
}

// Single-component check — used by detail routes.
export async function canAccess(componentName, accountName, env) {
  if (await isSuperAdmin(accountName, env)) return true;
  const entry = await getCdsAuthClient(env).get(componentName);
  return entry?.jomax?.includes(accountName) ?? false;
}

export async function assertCanAccess(componentName, accountName, env) {
  if (!(await canAccess(componentName, accountName, env))) {
    const err = new Error(`${accountName} does not have access to ${componentName}`);
    err.status = 403;
    throw err;
  }
}

// List-filtering variant — returns only components the user can access.
// For super admins, returns the full list unchanged. For regular users,
// does ONE cds-auth.get() call to get all entries, then filters in-memory.
export async function listAccessibleComponents(accountName, env, allComponentNames) {
  if (await isSuperAdmin(accountName, env)) return allComponentNames;
  const allAuth = await getCdsAuthClient(env).get() ?? {};
  return allComponentNames.filter(name => allAuth[name]?.jomax?.includes(accountName));
}
```

- [ ] **Step 5: Create middleware/require-same-origin.js** (CSRF defense)

```js
// middleware/require-same-origin.js
// State-changing requests must come from our own origin.
// Relies on SameSite cookies + Origin header check (adequate for internal tools).
const SAFE_METHODS = new Set(['GET', 'HEAD', 'OPTIONS']);

export function requireSameOrigin(req, res, next) {
  if (SAFE_METHODS.has(req.method)) return next();
  const origin = req.headers.origin || req.headers.referer;
  const expected = `https://${process.env.HOSTNAME || 'cds-admin.int.test-gdcorp.tools'}`;
  if (!origin || !origin.startsWith(expected)) {
    return res.status(403).json({ error: 'Origin mismatch' });
  }
  next();
}
```

- [ ] **Step 6: Create middleware/error-handler.js**

```js
// middleware/error-handler.js
// Express final error handler — must be registered AFTER all routes.
export function errorHandler(err, req, res, next) {
  const status = err.status || 500;
  // Don't leak stack traces in responses
  console.error('[cds-admin error]', err);
  res.status(status).json({ error: err.message || 'Internal server error' });
}
```

- [ ] **Step 7: Create routes/auth.js**

```js
// routes/auth.js
import express from 'express';
import { requireJomax } from '../middleware/require-jomax.js';

const router = express.Router();

router.get('/auth/me', requireJomax, (req, res) => {
  res.json({ accountName: req.user.accountName });
});

router.post('/auth/logout', (req, res) => {
  req.session?.destroy?.();
  res.json({ ok: true });
});

export default router;
```

- [ ] **Step 8: Create routes/config.js** (public non-secret config for the browser)

```js
// routes/config.js
// Exposes non-sensitive per-env config to the client. No auth required —
// these are just URLs the browser needs to build Kibana deep-links.
import express from 'express';

const router = express.Router();

router.get('/api/config', (req, res) => {
  const { kibanaDashboards = {}, kibanaDiscover = {} } = req.app.config ?? {};
  res.json({ kibanaDashboards, kibanaDiscover });
});

export default router;
```

Write the test:

```js
// test/routes/config.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const { default: router } = await import('../../routes/config.js');
const app = express();
app.config = {
  kibanaDashboards: {
    logs: 'https://logs.kb.example.com/app/dashboards#/view/abc',
    rum:  'https://rum.kb.example.com/app/dashboards#/view/xyz'
  },
  kibanaDiscover: {
    logsUrl: 'https://usiessp-prod-usw2.kb.us-west-2.aws.found.io:9243',
    logsDataViewId: 'e43871e5-f314-4ca9-8eb9-697e1799095c'
  }
};
app.use(router);

test('GET /api/config returns kibanaDashboards', async () => {
  const res = await request(app).get('/api/config');
  expect(res.status).toBe(200);
  expect(res.body.kibanaDashboards.logs).toContain('logs.kb.example.com');
  expect(res.body.kibanaDashboards.rum).toContain('rum.kb.example.com');
});

test('GET /api/config returns kibanaDiscover with logsUrl and logsDataViewId', async () => {
  const res = await request(app).get('/api/config');
  expect(res.status).toBe(200);
  expect(res.body.kibanaDiscover.logsUrl).toContain('usiessp-prod-usw2');
  expect(res.body.kibanaDiscover.logsDataViewId).toBe('e43871e5-f314-4ca9-8eb9-697e1799095c');
});

test('GET /api/config returns empty objects when not configured', async () => {
  const bare = express();
  bare.config = {};
  bare.use(router);
  const res = await request(bare).get('/api/config');
  expect(res.status).toBe(200);
  expect(res.body.kibanaDashboards).toEqual({});
  expect(res.body.kibanaDiscover).toEqual({});
});
```

- [ ] **Step 9: Wire middleware in gasket.js**

Add a plugin hook that registers `requireSameOrigin` globally and `errorHandler` as the final handler. Also attach `gasket.config` to `app.config` so `routes/config.js` can read `kibanaDashboards`:

```js
// Add to gasket.js plugins array:
{
  name: 'cds-admin-global-middleware',
  hooks: {
    express(gasket, app) {
      // Expose gasket config on Express app so routes can read kibanaDashboards etc.
      app.config = gasket.config;
    },
    async middleware() {
      const { requireSameOrigin } = await import('./middleware/require-same-origin.js');
      return [requireSameOrigin];
    },
    async errorMiddleware() {
      const { errorHandler } = await import('./middleware/error-handler.js');
      return [errorHandler];
    }
  }
}
```

- [ ] **Step 10: Run tests — expect PASS**

```bash
npm test -- test/middleware test/lib test/routes/config.test.js
```

- [ ] **Step 11: Commit**

```bash
git add middleware/ lib/ routes/auth.js routes/config.js test/middleware/ test/lib/ test/routes/config.test.js gasket.js
git commit -m "feat: jomax auth, allowlist, csrf, error handler, public config route"
```

---

## Task 4: Component List API + Page

**Files:**
- Create: `routes/components.js` (list + create endpoints only)
- Create: `test/routes/components.test.js`
- Modify: `pages/index.js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/components.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockCdsGet = jest.fn();
const mockCdsAuthGet = jest.fn();
const mockCdsAdminGet = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockCdsGet, put: jest.fn() })),
  getCdsAuthClient: jest.fn(() => ({ get: mockCdsAuthGet, put: jest.fn() })),
  getCdsAdminClient: jest.fn(() => ({ get: mockCdsAdminGet, put: jest.fn() }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/components.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components returns only components the user is allowlisted on with lastUpdated', async () => {
  mockCdsGet.mockResolvedValue({
    'registry.componentA': { metaData: { updatedAt: '2026-01-01T00:00:00Z' } },
    'registry.componentB': { metaData: { updatedAt: '2026-01-02T00:00:00Z' } },
    'registry.componentC': { metaData: { updatedAt: null } },
    'registry.componentA.rum': {},        // sub-keys excluded
    'registry.componentA.i18nVersion': {}
  });
  mockCdsAdminGet.mockResolvedValue({ jomax: [] });  // not a super admin
  mockCdsAuthGet.mockResolvedValue({
    componentA: { jomax: ['jroig'] },
    componentB: { jomax: ['alice'] },       // jroig NOT in here
    componentC: { jomax: ['jroig', 'bob'] }
  });

  const res = await request(app).get('/api/components');
  expect(res.status).toBe(200);
  expect(res.body).toEqual([
    { name: 'componentA', lastUpdated: '2026-01-01T00:00:00Z' },
    { name: 'componentC', lastUpdated: null }
  ]);
});

test('GET /api/components returns ALL components for super admins', async () => {
  mockCdsGet.mockResolvedValue({
    'registry.componentA': { metaData: { updatedAt: '2026-01-01T00:00:00Z' } },
    'registry.componentB': { metaData: {} }
  });
  mockCdsAdminGet.mockResolvedValue({ jomax: ['jroig'] });  // super admin

  const res = await request(app).get('/api/components');
  expect(res.status).toBe(200);
  expect(res.body).toEqual([
    { name: 'componentA', lastUpdated: '2026-01-01T00:00:00Z' },
    { name: 'componentB', lastUpdated: null }
  ]);
});

test('POST /api/components creates cds + cds-auth + cds-admin entries', async () => {
  const cdsClient = { get: jest.fn().mockResolvedValue({}), put: jest.fn().mockResolvedValue() };
  const authClient = { put: jest.fn().mockResolvedValue() };
  const adminClient = { put: jest.fn().mockResolvedValue() };
  const { getCdsClient, getCdsAuthClient, getCdsAdminClient } = await import('../../clients/switchboard.js');
  getCdsClient.mockReturnValue(cdsClient);
  getCdsAuthClient.mockReturnValue(authClient);
  getCdsAdminClient.mockReturnValue(adminClient);

  const res = await request(app)
    .post('/api/components')
    .send({ name: 'newComponent', env: 'dev' });

  expect(res.status).toBe(201);
  expect(cdsClient.put).toHaveBeenCalledWith('registry.newComponent', expect.any(Object));
  expect(authClient.put).toHaveBeenCalledWith('newComponent', { jomax: [''], cert: [''], awsiam: [''] });
  expect(adminClient.put).toHaveBeenCalledWith('config.newComponent', { slackWebhookUrl: null });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/routes/components.test.js
```

- [ ] **Step 3: Implement routes/components.js (list + create)**

```js
// routes/components.js
import express from 'express';
import { getCdsClient, getCdsAuthClient, getCdsAdminClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess, listAccessibleComponents } from '../lib/access-control.js';

const router = express.Router();

// Strip sub-keys (.rum, .i18nVersion, .notes) — only top-level registry.* keys
// Returns [{ name, lastUpdated }] sorted alphabetically
function extractComponents(allKeys) {
  return Object.entries(allKeys)
    .filter(([k]) => k.startsWith('registry.') && k.split('.').length === 2)
    .map(([k, v]) => ({
      name: k.replace('registry.', ''),
      lastUpdated: v?.metaData?.updatedAt ?? null
    }));
}

router.get('/api/components', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const all = await getCdsClient(env).get();
    const allComponents = extractComponents(all ?? {});
    const allNames = allComponents.map(c => c.name);
    // Filter to components the caller can access (super admins see all)
    const accessibleNames = await listAccessibleComponents(req.user.accountName, env, allNames);
    const accessible = allComponents.filter(c => accessibleNames.includes(c.name));
    res.json(accessible);
  } catch (err) { next(err); }
});

router.post('/api/components', requireJomax, async (req, res, next) => {
  try {
    const { name, env = 'dev' } = req.body;
    if (!name || !/^[a-zA-Z][a-zA-Z0-9]*$/.test(name)) {
      return res.status(400).json({ error: 'name must be camelCase alphanumeric' });
    }
    const stub = { dependencies: {}, engine: 'esm', js: '', metaData: {
      identity: 'unknown', updatedAt: new Date().toISOString()
    }};
    await Promise.all([
      getCdsClient(env).put(`registry.${name}`, stub),
      getCdsAuthClient(env).put(name, { jomax: [''], cert: [''], awsiam: [''] }),
      getCdsAdminClient(env).put(`config.${name}`, { slackWebhookUrl: null })
    ]);
    res.status(201).json({ name });
  } catch (err) { next(err); }
});

export default router;
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npm test -- test/routes/components.test.js
```

- [ ] **Step 5: Implement pages/index.js**

```jsx
// pages/index.js
// Alphabetically sorted, case-insensitive search. Shows lastUpdated per component.
import { useState, useEffect, useMemo } from 'react';
import Layout from '../components/Layout.js';

function formatDate(iso) {
  if (!iso) return '—';
  return new Date(iso).toLocaleString(undefined, { dateStyle: 'short', timeStyle: 'short' });
}

export default function ComponentList() {
  const [components, setComponents] = useState([]);
  const [env, setEnv] = useState('dev');
  const [query, setQuery] = useState('');

  useEffect(() => {
    fetch(`/api/components?env=${env}`)
      .then(r => r.json())
      .then(setComponents);
  }, [env]);

  const sorted = useMemo(
    () => [...components].sort((a, b) => a.name.localeCompare(b.name, undefined, { sensitivity: 'base' })),
    [components]
  );

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return sorted;
    return sorted.filter(({ name }) => name.toLowerCase().includes(q));
  }, [sorted, query]);

  return (
    <Layout env={env} onEnvChange={setEnv}>
      <div style={{ padding: 24, maxWidth: 800, margin: '0 auto' }}>
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 }}>
          <h1 style={{ margin: 0 }}>Components</h1>
          <a
            href="/components/new"
            style={{ padding: '8px 16px', background: '#0070d2', color: '#fff', borderRadius: 4, textDecoration: 'none' }}
          >
            + New Component
          </a>
        </div>

        <input
          type="search"
          autoFocus
          placeholder="Search components…"
          value={query}
          onChange={e => setQuery(e.target.value)}
          style={{ width: '100%', padding: '10px 14px', fontSize: 16, border: '1px solid #ccc', borderRadius: 6, marginBottom: 16 }}
        />

        <div style={{ fontSize: 12, color: '#666', marginBottom: 8 }}>
          {filtered.length} of {sorted.length} components
        </div>

        <table style={{ width: '100%', borderCollapse: 'collapse' }}>
          <thead>
            <tr style={{ textAlign: 'left', borderBottom: '2px solid #eee', fontSize: 12, color: '#666' }}>
              <th style={{ padding: '6px 8px' }}>Name</th>
              <th style={{ padding: '6px 8px' }}>Last updated</th>
            </tr>
          </thead>
          <tbody>
            {filtered.map(({ name, lastUpdated }) => (
              <tr key={name} style={{ borderBottom: '1px solid #eee' }}>
                <td style={{ padding: '10px 8px' }}>
                  <a href={`/components/${name}`} style={{ color: '#0070d2', textDecoration: 'none' }}>{name}</a>
                </td>
                <td style={{ padding: '10px 8px', fontSize: 12, color: '#666' }}>{formatDate(lastUpdated)}</td>
              </tr>
            ))}
            {filtered.length === 0 && (
              <tr><td colSpan={2} style={{ padding: 16, color: '#999', textAlign: 'center' }}>
                No components match "{query}"
              </td></tr>
            )}
          </tbody>
        </table>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 6: Create components/Layout.js**

```jsx
// components/Layout.js
export default function Layout({ children, env, onEnvChange }) {
  return (
    <div>
      <nav style={{ background: '#1b1b1b', color: '#fff', padding: '12px 24px', display: 'flex', alignItems: 'center', gap: 16 }}>
        <a href="/" style={{ color: '#fff', fontWeight: 'bold', textDecoration: 'none' }}>CDS Admin</a>
        <a href="/analytics" style={{ color: '#aaa', textDecoration: 'none' }}>Analytics</a>
        {onEnvChange && (
          <select
            value={env}
            onChange={e => onEnvChange(e.target.value)}
            style={{ marginLeft: 'auto', padding: '4px 8px' }}
          >
            <option value="dev">dev</option>
            <option value="test">test</option>
            <option value="prod">prod</option>
          </select>
        )}
      </nav>
      <main>{children}</main>
    </div>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add routes/components.js pages/index.js components/Layout.js test/routes/components.test.js
git commit -m "feat: component list API and page"
```

---

## Task 5: Create Component Page

**Files:**
- Create: `pages/components/new.js`

- [ ] **Step 1: Implement pages/components/new.js**

```jsx
// pages/components/new.js
import { useState } from 'react';
import { useRouter } from 'next/router.js';
import Layout from '../../components/Layout.js';

export default function NewComponent() {
  const router = useRouter();
  const [name, setName] = useState('');
  const [env, setEnv] = useState('dev');
  const [error, setError] = useState('');
  const [saving, setSaving] = useState(false);

  async function handleSubmit(e) {
    e.preventDefault();
    setError('');
    setSaving(true);
    const res = await fetch('/api/components', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ name, env })
    });
    setSaving(false);
    if (res.ok) {
      router.push(`/components/${name}/manifest`);
    } else {
      const body = await res.json();
      setError(body.error || 'Failed to create component');
    }
  }

  return (
    <Layout env={env} onEnvChange={setEnv}>
      <div style={{ padding: 24, maxWidth: 480 }}>
        <a href="/">← Back</a>
        <h1>New Component</h1>
        <form onSubmit={handleSubmit}>
          <label>
            Component name (camelCase)
            <input
              value={name}
              onChange={e => setName(e.target.value)}
              placeholder="myComponentEsm"
              pattern="[a-zA-Z][a-zA-Z0-9]*"
              required
              style={{ display: 'block', width: '100%', marginTop: 4 }}
            />
          </label>
          {error && <p style={{ color: 'red' }}>{error}</p>}
          <button type="submit" disabled={saving} style={{ marginTop: 16 }}>
            {saving ? 'Creating…' : 'Create Component'}
          </button>
        </form>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 2: Verify in browser**

```bash
npm run local
```

Navigate to `http://localhost:3000/components/new`, create a test component, confirm redirect to detail page (404 for now, that's ok).

- [ ] **Step 3: Commit**

```bash
git add pages/components/new.js
git commit -m "feat: create component page"
```

---

## Task 6: Manifest Get/Edit API + JsonEditor Component

**Files:**
- Add to `routes/components.js`: GET + PUT `:name`
- Create: `components/JsonEditor.js`

- [ ] **Step 1: Add tests for get/put manifest**

Add to `test/routes/components.test.js`:

```js
test('GET /api/components/:name returns manifest', async () => {
  const manifest = { engine: 'esm', js: 'abc.es.js', dependencies: {} };
  mockGet.mockResolvedValue(manifest);
  const res = await request(app).get('/api/components/componentA?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual(manifest);
});

test('PUT /api/components/:name updates manifest (allowlisted user)', async () => {
  const { getCdsAuthClient, getCdsClient } = await import('../../clients/switchboard.js');
  getCdsAuthClient.mockReturnValue({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) });
  const mockPut = jest.fn().mockResolvedValue();
  getCdsClient.mockReturnValue({ put: mockPut });

  const manifest = { engine: 'esm', js: 'new.es.js', dependencies: {} };
  const res = await request(app)
    .put('/api/components/componentA?env=dev')
    .send(manifest);

  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('registry.componentA', manifest);
});

test('PUT /api/components/:name returns 403 for non-allowlisted user', async () => {
  const { getCdsAuthClient } = await import('../../clients/switchboard.js');
  getCdsAuthClient.mockReturnValue({ get: jest.fn().mockResolvedValue({ jomax: ['alice'] }) });

  const res = await request(app)
    .put('/api/components/componentA?env=dev')
    .send({ engine: 'esm' });

  expect(res.status).toBe(403);
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/routes/components.test.js
```

- [ ] **Step 3: Add GET + PUT to routes/components.js**

```js
// Add inside routes/components.js after the POST route:

router.get('/api/components/:name', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    // Gate reads on allowlist/super-admin — users can only see components they have access to
    await assertCanAccess(req.params.name, req.user.accountName, env);
    const manifest = await getCdsClient(env).get(`registry.${req.params.name}`);
    if (!manifest) return res.status(404).json({ error: 'Component not found' });
    res.json(manifest);
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.put('/api/components/:name', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}`, req.body);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
npm test -- test/routes/components.test.js
```

- [ ] **Step 5: Create components/JsonEditor.js**

```jsx
// components/JsonEditor.js
// Cleanly separates value-change from validation-change to avoid stale-closure bugs.
// Parent passes `onChange` (value updates) and `onValidate` (valid/invalid flag) independently.
import dynamic from 'next/dynamic';
import { useState } from 'react';

const MonacoEditor = dynamic(() => import('@monaco-editor/react'), { ssr: false });

export default function JsonEditor({ value, onChange, onValidate, readOnly = false, height = '400px' }) {
  const [hasErrors, setHasErrors] = useState(false);

  function handleValidation(markers) {
    const invalid = markers.length > 0;
    setHasErrors(invalid);
    onValidate?.(!invalid);
  }

  return (
    <div>
      {hasErrors && (
        <div style={{ background: '#fee', color: '#c00', padding: '4px 8px', fontSize: 12 }}>
          Invalid JSON — fix errors before saving
        </div>
      )}
      <MonacoEditor
        height={height}
        defaultLanguage="json"
        value={typeof value === 'string' ? value : JSON.stringify(value, null, 2)}
        onChange={onChange}
        onValidate={handleValidation}
        options={{ readOnly, minimap: { enabled: false }, scrollBeyondLastLine: false }}
      />
    </div>
  );
}
```

- [ ] **Step 6: Verify API in browser** — use `curl` or the browser to confirm GET/PUT work against the local dev server.

- [ ] **Step 7: Commit**

```bash
git add routes/components.js components/JsonEditor.js test/routes/components.test.js
git commit -m "feat: manifest GET/PUT API + JsonEditor component"
```

---

## Task 7: History + Rollback API + HistoryTimeline Component

**Files:**
- Add to `routes/components.js`: GET `:name/history`, POST `:name/rollback`
- Create: `components/HistoryTimeline.js`

- [ ] **Step 1: Add tests**

Add to `test/routes/components.test.js`:

```js
test('GET /api/components/:name/history returns history array', async () => {
  const { getCdsClient } = await import('../../clients/switchboard.js');
  const mockHistory = [{ value: { engine: 'esm' }, updatedAt: '2026-01-01T00:00:00Z' }];
  getCdsClient.mockReturnValue({ getHistory: jest.fn().mockResolvedValue(mockHistory) });

  const res = await request(app).get('/api/components/componentA/history?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual(mockHistory);
});

test('POST /api/components/:name/rollback re-puts historical value', async () => {
  const { getCdsAuthClient, getCdsClient } = await import('../../clients/switchboard.js');
  getCdsAuthClient.mockReturnValue({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) });
  const mockPut = jest.fn().mockResolvedValue();
  getCdsClient.mockReturnValue({ put: mockPut });

  const oldManifest = { engine: 'esm', js: 'old.es.js', dependencies: {} };
  const res = await request(app)
    .post('/api/components/componentA/rollback?env=dev')
    .send({ value: oldManifest });

  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('registry.componentA', oldManifest);
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/routes/components.test.js
```

- [ ] **Step 3: Add history + rollback routes to routes/components.js**

```js
router.get('/api/components/:name/history', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const history = await getCdsClient(env).getHistory(`registry.${req.params.name}`);
    res.json(history ?? []);
  } catch (err) { next(err); }
});

router.post('/api/components/:name/rollback', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create components/HistoryTimeline.js**

```jsx
// components/HistoryTimeline.js
import { useState } from 'react';

export default function HistoryTimeline({ history, onRollback, loading }) {
  const [rolling, setRolling] = useState(null);

  async function handleRollback(entry, idx) {
    if (!confirm(`Roll back to version from ${new Date(entry.updatedAt).toLocaleString()}?`)) return;
    setRolling(idx);
    await onRollback(entry.value);
    setRolling(null);
  }

  if (loading) return <p>Loading history…</p>;
  if (!history?.length) return <p>No history available.</p>;

  return (
    <div>
      {history.map((entry, idx) => (
        <div key={idx} style={{ borderBottom: '1px solid #eee', padding: '12px 0', display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
          <div>
            <div style={{ fontWeight: 'bold' }}>{new Date(entry.updatedAt).toLocaleString()}</div>
            <div style={{ color: '#666', fontSize: 12 }}>{entry.updatedBy || 'unknown'}</div>
          </div>
          <button
            onClick={() => handleRollback(entry, idx)}
            disabled={rolling === idx}
            style={{ fontSize: 12 }}
          >
            {rolling === idx ? 'Rolling back…' : 'Rollback to this version'}
          </button>
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add routes/components.js components/HistoryTimeline.js
git commit -m "feat: manifest history/rollback API + HistoryTimeline component"
```

---

## Task 8: ComponentNav + Component Main Dashboard Page

**Files:**
- Create: `components/ComponentNav.js`
- Create: `pages/components/[name]/index.js`

The main page shows RUM sparklines + error sparklines + a notes textarea. It fetches the manifest only to read `metaData.updatedAt` for the overview header — the full editor lives on the Manifest sub-page. Sparklines come from the analytics API. The notes textarea auto-saves.

`ComponentNav` is the tab bar reused on every sub-page. It reads the current path to highlight the active tab. Tab links include the current `env` query param so switching tabs preserves the selected environment.

- [ ] **Step 1: Create components/ComponentNav.js**

```jsx
// components/ComponentNav.js
// Shared tab bar for all /components/[name]/* pages.
// Tab links carry the current env query param so switching tabs preserves the environment.
import { useRouter } from 'next/router';

const TABS = [
  { label: 'Overview',  href: '' },         // /components/[name]
  { label: 'Manifest',  href: '/manifest' },
  { label: 'Auth',      href: '/auth' },
  { label: 'RUM',       href: '/rum' },
  { label: 'Logs',      href: '/logs' },
  { label: 'Sync',      href: '/sync' },
];

export default function ComponentNav({ name, env }) {
  const router = useRouter();
  const base = `/components/${name}`;

  return (
    <nav style={{ borderBottom: '1px solid #e5e5e5', marginBottom: 24, display: 'flex', gap: 0, alignItems: 'center' }}>
      <span style={{ fontSize: 14, fontWeight: 600, marginRight: 24, color: '#333' }}>{name}</span>
      {TABS.map(({ label, href }) => {
        const url = `${base}${href}`;
        const active = router.asPath.startsWith(url + '?') || router.asPath === url ||
          (href === '' && router.asPath === base);
        return (
          <a
            key={label}
            href={`${url}?env=${env}`}
            style={{
              padding: '10px 16px',
              textDecoration: 'none',
              borderBottom: active ? '2px solid #0070d2' : '2px solid transparent',
              color: active ? '#0070d2' : '#555',
              fontWeight: active ? 600 : 400,
              fontSize: 14
            }}
          >
            {label}
          </a>
        );
      })}
    </nav>
  );
}
```

- [ ] **Step 2: Create pages/components/[name]/index.js**

```jsx
// pages/components/[name]/index.js
// Main dashboard: RUM sparklines + error sparklines + notes textarea.
// Fetches the manifest only for lastUpdated; full editing is on the Manifest sub-page.
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import Layout from '../../../components/Layout.js';
import ComponentNav from '../../../components/ComponentNav.js';
import Notes from '../../../components/Notes.js';
import MiniChart from '../../../components/charts/MiniChart.js';

export default function ComponentMain() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState(router.query.env || 'dev');
  const [manifest, setManifest] = useState(null);
  const [errors, setErrors] = useState({ total: 0, buckets: [] });
  const [volume, setVolume] = useState({ buckets: [] });
  const [vitals, setVitals] = useState({ buckets: [] });

  useEffect(() => {
    if (!name) return;
    fetch(`/api/components/${name}?env=${env}`).then(r => r.json()).then(setManifest);
    const p = `component=${name}&from=now-1d&to=now`;
    fetch(`/api/analytics/errors?${p}`).then(r => r.json()).then(setErrors);
    fetch(`/api/analytics/volume?${p}`).then(r => r.json()).then(setVolume);
    fetch(`/api/analytics/vitals?${p}`).then(r => r.json()).then(setVitals);
  }, [name, env]);

  function handleEnvChange(e) {
    setEnv(e);
    router.replace({ query: { ...router.query, env: e } });
  }

  return (
    <Layout env={env} onEnvChange={handleEnvChange}>
      <div style={{ padding: 24, maxWidth: 1200, margin: '0 auto' }}>
        <a href="/" style={{ fontSize: 12, color: '#666' }}>← All components</a>
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline', margin: '8px 0 16px' }}>
          <div>
            <h1 style={{ margin: 0 }}>{name}</h1>
            {manifest?.metaData?.updatedAt && (
              <p style={{ margin: '4px 0 0', fontSize: 12, color: '#666' }}>
                Last updated: {new Date(manifest.metaData.updatedAt).toLocaleString()} · {manifest.engine || '—'} · {manifest.js || '—'}
              </p>
            )}
          </div>
          <a href={`/analytics?component=${name}`} style={{ fontSize: 13, color: '#0070d2' }}>Full analytics →</a>
        </div>

        {name && <ComponentNav name={name} env={env} />}

        <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: 16, marginBottom: 32 }}>
          <MiniChart data={errors.buckets} dataKey="count" color="#e53e3e"
            label={`Errors — last 24h (${errors.total} total)`} />
          <MiniChart data={volume.buckets} dataKey="count" color="#38a169"
            label="Request volume — last 24h" />
          <MiniChart data={vitals.buckets} dataKey="lcp_p75" color="#3182ce"
            label="LCP p75 — last 24h" unit="ms" />
        </div>

        {name && <Notes componentName={name} env={env} />}
      </div>
    </Layout>
  );
}
```

- [ ] **Step 3: Verify in browser** — navigate to `/components/{name}`, confirm sparklines load, notes auto-save works, tab bar renders with env dropdown.

- [ ] **Step 4: Commit**

```bash
git add components/ComponentNav.js pages/components/
git commit -m "feat: ComponentNav + component main dashboard page"
```

---

## Task 9: Notes API + Notes Component

**Files:**
- Create: `routes/notes.js`
- Create: `test/routes/notes.test.js`
- Create: `components/Notes.js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/notes.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockCdsGet = jest.fn();
const mockCdsPut = jest.fn();
const mockCdsAuthGet = jest.fn().mockResolvedValue({ jomax: ['jroig'] });
const mockCdsAdminGet = jest.fn().mockResolvedValue({ jomax: [] });

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockCdsGet, put: mockCdsPut })),
  getCdsAuthClient: jest.fn(() => ({ get: mockCdsAuthGet })),
  getCdsAdminClient: jest.fn(() => ({ get: mockCdsAdminGet }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/notes.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components/:name/notes returns string', async () => {
  mockCdsGet.mockResolvedValue('# Deployment notes\n\nRolled back due to LCP regression.');
  const res = await request(app).get('/api/components/componentA/notes?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ value: '# Deployment notes\n\nRolled back due to LCP regression.' });
});

test('GET /api/components/:name/notes returns empty when unset', async () => {
  mockCdsGet.mockResolvedValue(null);
  const res = await request(app).get('/api/components/componentA/notes?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ value: '' });
});

test('PUT /api/components/:name/notes writes string', async () => {
  mockCdsPut.mockResolvedValue();
  const res = await request(app)
    .put('/api/components/componentA/notes?env=dev')
    .send({ value: '## New notes' });
  expect(res.status).toBe(200);
  expect(mockCdsPut).toHaveBeenCalledWith('registry.componentA.notes', '## New notes');
});

test('PUT /api/components/:name/notes rejects non-string body', async () => {
  const res = await request(app)
    .put('/api/components/componentA/notes?env=dev')
    .send({ value: { not: 'a string' } });
  expect(res.status).toBe(400);
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/routes/notes.test.js
```

- [ ] **Step 3: Create routes/notes.js**

```js
// routes/notes.js
import express from 'express';
import { getCdsClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/components/:name/notes', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    await assertCanAccess(req.params.name, req.user.accountName, env);
    const value = await getCdsClient(env).get(`registry.${req.params.name}.notes`);
    res.json({ value: value ?? '' });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.put('/api/components/:name/notes', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    if (typeof req.body.value !== 'string') {
      return res.status(400).json({ error: 'value must be a string' });
    }
    await getCdsClient(env).put(`registry.${name}.notes`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create components/Notes.js**

```jsx
// components/Notes.js
// Plain-text textarea with debounced auto-save (1.5s after typing stops).
import { useState, useEffect, useRef } from 'react';

export default function Notes({ componentName, env }) {
  const [value, setValue] = useState('');
  const [status, setStatus] = useState('');
  const [loaded, setLoaded] = useState(false);
  const saveTimer = useRef(null);

  useEffect(() => {
    fetch(`/api/components/${componentName}/notes?env=${env}`)
      .then(r => r.json())
      .then(data => { setValue(data.value ?? ''); setLoaded(true); });
  }, [componentName, env]);

  useEffect(() => {
    if (!loaded) return;
    clearTimeout(saveTimer.current);
    setStatus('editing…');
    saveTimer.current = setTimeout(async () => {
      setStatus('saving…');
      const res = await fetch(`/api/components/${componentName}/notes?env=${env}`, {
        method: 'PUT',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ value })
      });
      setStatus(res.ok ? 'saved' : 'save failed');
      setTimeout(() => setStatus(''), 1500);
    }, 1500);
    return () => clearTimeout(saveTimer.current);
  }, [value]);

  if (!loaded) return <p>Loading…</p>;

  return (
    <div>
      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 4 }}>
        <label style={{ fontSize: 13, fontWeight: 600, color: '#555' }}>Notes</label>
        <span style={{ fontSize: 11, color: '#999' }}>{status}</span>
      </div>
      <textarea
        value={value}
        onChange={e => setValue(e.target.value)}
        placeholder="Add deployment notes, known issues, anything the team should know…"
        style={{ width: '100%', minHeight: 160, fontFamily: 'monospace', fontSize: 13, padding: 8,
          border: '1px solid #ccc', borderRadius: 4, resize: 'vertical', boxSizing: 'border-box' }}
      />
    </div>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add routes/notes.js test/routes/notes.test.js components/Notes.js
git commit -m "feat: notes API + auto-save Notes component"
```

---

## Task 10: Manifest Sub-page

**Files:**
- Create: `pages/components/[name]/manifest.js`
- Create: `routes/i18n.js` (needed by this page)
- Create: `clients/slack.js`
- Create: `routes/alerts.js`

This page brings together everything manifest-related: the JSON editor, history/rollback, i18n version hash, and Slack alert config. It reuses `JsonEditor`, `HistoryTimeline`, and the alerts API. The i18n API and alerts API are implemented here because they're only used on this page.

- [ ] **Step 1: Create routes/i18n.js**

```js
// routes/i18n.js
import express from 'express';
import { getCdsClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/components/:name/i18n', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const value = await getCdsClient(env).get(`registry.${req.params.name}.i18nVersion`);
    if (value == null) return res.status(404).json({ error: 'i18n not configured' });
    res.json({ value });
  } catch (err) { next(err); }
});

router.put('/api/components/:name/i18n', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.i18nVersion`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.get('/api/components/:name/i18n/history', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const history = await getCdsClient(env).getHistory(`registry.${req.params.name}.i18nVersion`);
    res.json(history ?? []);
  } catch (err) { next(err); }
});

router.post('/api/components/:name/i18n/rollback', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.i18nVersion`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 2: Create clients/slack.js**

```js
// clients/slack.js
export async function sendSlackMessage(webhookUrl, text) {
  const res = await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ text })
  });
  if (!res.ok) throw new Error(`Slack webhook failed: ${res.status}`);
}
```

- [ ] **Step 3: Create routes/alerts.js**

```js
// routes/alerts.js
import express from 'express';
import { getCdsAdminClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';
import { sendSlackMessage } from '../clients/slack.js';

const router = express.Router();

router.get('/api/alerts/:name', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const config = await getCdsAdminClient(env).get(`config.${req.params.name}`);
    res.json(config ?? { slackWebhookUrl: null });
  } catch (err) { next(err); }
});

router.put('/api/alerts/:name', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsAdminClient(env).put(`config.${name}`, { slackWebhookUrl: req.body.slackWebhookUrl });
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.post('/api/alerts/:name/test', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const config = await getCdsAdminClient(env).get(`config.${req.params.name}`);
    if (!config?.slackWebhookUrl) return res.status(400).json({ error: 'No Slack webhook configured' });
    await sendSlackMessage(config.slackWebhookUrl, `Test message from CDS Admin for ${req.params.name}`);
    res.json({ ok: true });
  } catch (err) { next(err); }
});

export default router;
```

- [ ] **Step 4: Write failing tests for i18n and alerts routes**

```js
// test/routes/i18n.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();
const mockGetHistory = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockGet, put: mockPut, getHistory: mockGetHistory })),
  getCdsAuthClient: jest.fn(() => ({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/i18n.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components/:name/i18n returns hash string', async () => {
  mockGet.mockResolvedValue('607a011');
  const res = await request(app).get('/api/components/componentA/i18n?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ value: '607a011' });
});

test('GET /api/components/:name/i18n returns 404 when not set', async () => {
  mockGet.mockResolvedValue(null);
  const res = await request(app).get('/api/components/componentA/i18n?env=dev');
  expect(res.status).toBe(404);
});

test('PUT /api/components/:name/i18n updates hash', async () => {
  mockPut.mockResolvedValue();
  const res = await request(app)
    .put('/api/components/componentA/i18n?env=dev')
    .send({ value: 'abc1234' });
  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('registry.componentA.i18nVersion', 'abc1234');
});
```

```js
// test/routes/alerts.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsAdminClient: jest.fn(() => ({ get: mockGet, put: mockPut })),
  getCdsAuthClient: jest.fn(() => ({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) }))
}));
jest.unstable_mockModule('../../clients/slack.js', () => ({
  sendSlackMessage: jest.fn().mockResolvedValue()
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/alerts.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/alerts/:name returns slack config', async () => {
  mockGet.mockResolvedValue({ slackWebhookUrl: 'https://hooks.slack.com/test' });
  const res = await request(app).get('/api/alerts/componentA?env=dev');
  expect(res.status).toBe(200);
  expect(res.body.slackWebhookUrl).toBe('https://hooks.slack.com/test');
});

test('POST /api/alerts/:name/test sends a slack message', async () => {
  mockGet.mockResolvedValue({ slackWebhookUrl: 'https://hooks.slack.com/test' });
  const { sendSlackMessage } = await import('../../clients/slack.js');
  const res = await request(app).post('/api/alerts/componentA/test?env=dev');
  expect(res.status).toBe(200);
  expect(sendSlackMessage).toHaveBeenCalledWith('https://hooks.slack.com/test', expect.stringContaining('componentA'));
});
```

- [ ] **Step 5: Run tests — expect FAIL, then implement, then PASS**

```bash
npm test -- test/routes/i18n.test.js test/routes/alerts.test.js
```

- [ ] **Step 6: Create pages/components/[name]/manifest.js**

```jsx
// pages/components/[name]/manifest.js
// Manifest editor + history/rollback + i18n version + Slack alert config.
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import Layout from '../../../components/Layout.js';
import ComponentNav from '../../../components/ComponentNav.js';
import JsonEditor from '../../../components/JsonEditor.js';
import HistoryTimeline from '../../../components/HistoryTimeline.js';

export default function ManifestPage() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState(router.query.env || 'dev');
  const [manifest, setManifest] = useState(null);
  const [editorValue, setEditorValue] = useState('');
  const [editorValid, setEditorValid] = useState(true);
  const [saving, setSaving] = useState(false);
  const [saveMsg, setSaveMsg] = useState('');
  const [history, setHistory] = useState([]);
  const [i18n, setI18n] = useState('');
  const [webhook, setWebhook] = useState('');

  useEffect(() => {
    if (!name) return;
    fetch(`/api/components/${name}?env=${env}`).then(r => r.json()).then(m => {
      setManifest(m);
      setEditorValue(JSON.stringify(m, null, 2));
    });
    fetch(`/api/components/${name}/history?env=${env}`).then(r => r.json()).then(setHistory);
    fetch(`/api/components/${name}/i18n?env=${env}`).then(r => r.ok ? r.json() : { value: '' }).then(d => setI18n(d.value ?? ''));
    fetch(`/api/alerts/${name}?env=${env}`).then(r => r.json()).then(d => setWebhook(d.slackWebhookUrl ?? ''));
  }, [name, env]);

  async function saveManifest() {
    setSaving(true);
    setSaveMsg('');
    const res = await fetch(`/api/components/${name}?env=${env}`, {
      method: 'PUT', headers: { 'content-type': 'application/json' }, body: editorValue
    });
    setSaving(false);
    setSaveMsg(res.ok ? 'Saved' : 'Error saving');
    if (res.ok) fetch(`/api/components/${name}/history?env=${env}`).then(r => r.json()).then(setHistory);
  }

  const sectionStyle = { marginBottom: 32 };
  const headingStyle = { fontSize: 14, textTransform: 'uppercase', letterSpacing: 0.5, color: '#666', marginBottom: 12 };

  function handleEnvChange(e) {
    setEnv(e);
    router.replace({ query: { ...router.query, env: e } });
  }

  return (
    <Layout env={env} onEnvChange={handleEnvChange}>
      <div style={{ padding: 24, maxWidth: 1200, margin: '0 auto' }}>
        <a href="/" style={{ fontSize: 12, color: '#666' }}>← All components</a>
        <h1 style={{ margin: '8px 0 16px' }}>{name}</h1>
        {name && <ComponentNav name={name} env={env} />}

        <section style={sectionStyle}>
          <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 8 }}>
            <h3 style={headingStyle}>Manifest</h3>
            <div>
              {saveMsg && <span style={{ marginRight: 8, fontSize: 12, color: '#666' }}>{saveMsg}</span>}
              <button onClick={saveManifest} disabled={!editorValid || saving}>
                {saving ? 'Saving…' : 'Save Manifest'}
              </button>
            </div>
          </div>
          <JsonEditor value={editorValue} onChange={v => setEditorValue(v ?? '')}
            onValidate={setEditorValid} height="400px" />
        </section>

        <section style={sectionStyle}>
          <h3 style={headingStyle}>History</h3>
          <HistoryTimeline history={history} onRollback={async value => {
            await fetch(`/api/components/${name}/rollback?env=${env}`, {
              method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ value })
            });
            const m = await fetch(`/api/components/${name}?env=${env}`).then(r => r.json());
            setManifest(m); setEditorValue(JSON.stringify(m, null, 2));
            fetch(`/api/components/${name}/history?env=${env}`).then(r => r.json()).then(setHistory);
          }} />
        </section>

        <section style={{ ...sectionStyle, display: 'grid', gridTemplateColumns: '1fr 1fr', gap: 24 }}>
          <div>
            <h3 style={headingStyle}>i18n Version Hash</h3>
            <input value={i18n} onChange={e => setI18n(e.target.value)}
              placeholder="e.g. 607a011" style={{ width: '100%', marginBottom: 8 }} />
            <button onClick={async () => {
              await fetch(`/api/components/${name}/i18n?env=${env}`, {
                method: 'PUT', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ value: i18n })
              });
            }}>Save i18n</button>
          </div>
          <div>
            <h3 style={headingStyle}>Slack Alerts</h3>
            <input value={webhook} onChange={e => setWebhook(e.target.value)}
              placeholder="https://hooks.slack.com/…" style={{ width: '100%', marginBottom: 8 }} />
            <button onClick={async () => {
              await fetch(`/api/alerts/${name}?env=${env}`, {
                method: 'PUT', headers: { 'content-type': 'application/json' },
                body: JSON.stringify({ slackWebhookUrl: webhook })
              });
            }}>Save Webhook</button>
            <button onClick={() => fetch(`/api/alerts/${name}/test?env=${env}`, { method: 'POST' })}
              disabled={!webhook} style={{ marginLeft: 8 }}>
              Send test message
            </button>
          </div>
        </section>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 7: Verify in browser** — navigate to `/components/{name}/manifest`, edit JSON, confirm save and rollback work, i18n hash saves, Slack test fires.

- [ ] **Step 8: Commit**

```bash
git add routes/i18n.js routes/alerts.js clients/slack.js \
  test/routes/i18n.test.js test/routes/alerts.test.js \
  pages/components/
git commit -m "feat: manifest sub-page with editor, history, i18n, and Slack alerts"
```

---

## Task 11: Access API + AccessList Component

**Files:**
- Create: `routes/access.js`
- Create: `test/routes/access.test.js`
- Create: `components/AccessList.js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/access.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsAuthClient: jest.fn(() => ({ get: mockGet, put: mockPut }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/access.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/access/:name returns allowlist entry', async () => {
  mockGet.mockResolvedValue({ jomax: ['jroig'], cert: [''], awsiam: [''] });
  const res = await request(app).get('/api/access/componentA?env=dev');
  expect(res.status).toBe(200);
  expect(res.body.jomax).toEqual(['jroig']);
});

test('PUT /api/access/:name updates jomax array', async () => {
  mockGet.mockResolvedValue({ jomax: ['jroig'], cert: [''], awsiam: [''] });
  mockPut.mockResolvedValue();
  const res = await request(app)
    .put('/api/access/componentA?env=dev')
    .send({ jomax: ['jroig', 'alice'] });
  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('componentA', { jomax: ['jroig', 'alice'], cert: [''], awsiam: [''] });
});

test('PUT /api/access/:name rejects empty jomax array (lockout prevention)', async () => {
  mockGet.mockResolvedValue({ jomax: ['jroig'], cert: [''], awsiam: [''] });
  const res = await request(app)
    .put('/api/access/componentA?env=dev')
    .send({ jomax: [] });
  expect(res.status).toBe(400);
  expect(mockPut).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create routes/access.js**

```js
// routes/access.js
import express from 'express';
import { getCdsAuthClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/access/:name', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const entry = await getCdsAuthClient(env).get(req.params.name);
    if (!entry) return res.status(404).json({ error: 'Component access entry not found' });
    res.json(entry);
  } catch (err) { next(err); }
});

router.put('/api/access/:name', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    const newJomax = Array.isArray(req.body.jomax) ? req.body.jomax : [];
    if (newJomax.length === 0) {
      return res.status(400).json({ error: 'Allowlist cannot be empty (at least one jomax user required)' });
    }
    const current = await getCdsAuthClient(env).get(name) ?? { jomax: [], cert: [''], awsiam: [''] };
    await getCdsAuthClient(env).put(name, { ...current, jomax: newJomax });
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create components/AccessList.js**

```jsx
// components/AccessList.js
// Shows jomax, cert, awsiam sections. Only jomax is editable through the UI.
// cert and awsiam are shown read-only (edit via direct Switchboard access or CLI).
import { useState, useEffect } from 'react';

export default function AccessList({ componentName, env }) {
  const [entry, setEntry] = useState(null);
  const [newUser, setNewUser] = useState('');
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  useEffect(() => {
    fetch(`/api/access/${componentName}?env=${env}`).then(r => r.json()).then(setEntry);
  }, [componentName, env]);

  async function saveJomax(newJomax) {
    setSaving(true);
    setError('');
    const res = await fetch(`/api/access/${componentName}?env=${env}`, {
      method: 'PUT',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ jomax: newJomax })
    });
    setSaving(false);
    if (res.ok) {
      setEntry(e => ({ ...e, jomax: newJomax }));
    } else {
      const body = await res.json();
      setError(body.error || 'Failed to update');
    }
  }

  if (!entry) return <p>Loading…</p>;

  const canRemove = entry.jomax.filter(u => u !== '').length > 1;

  return (
    <div>
      {error && <p style={{ color: 'red' }}>{error}</p>}

      <h4>Jomax users</h4>
      <ul style={{ margin: '0 0 8px', padding: 0, listStyle: 'none' }}>
        {entry.jomax.filter(u => u !== '').map(user => (
          <li key={user} style={{ marginBottom: 4 }}>
            {user}
            <button
              onClick={() => saveJomax(entry.jomax.filter(u => u !== user))}
              disabled={saving || !canRemove}
              title={!canRemove ? 'Cannot remove the last user' : ''}
              style={{ marginLeft: 8, fontSize: 11 }}
            >
              Remove
            </button>
          </li>
        ))}
      </ul>
      <div style={{ display: 'flex', gap: 8 }}>
        <input value={newUser} onChange={e => setNewUser(e.target.value)} placeholder="jomax username" />
        <button onClick={() => { saveJomax([...entry.jomax.filter(u => u !== ''), newUser]); setNewUser(''); }}
          disabled={saving || !newUser}>
          Add
        </button>
      </div>

      {entry.cert?.filter(c => c !== '').length > 0 && (
        <div style={{ marginTop: 16 }}>
          <h4>Cert (read-only — edit via Switchboard or CLI)</h4>
          <ul>{entry.cert.filter(c => c !== '').map(c => <li key={c} style={{ fontFamily: 'monospace', fontSize: 12 }}>{c}</li>)}</ul>
        </div>
      )}

      {entry.awsiam?.filter(a => a !== '').length > 0 && (
        <div style={{ marginTop: 16 }}>
          <h4>AWS IAM (read-only — edit via Switchboard or CLI)</h4>
          <ul>{entry.awsiam.filter(a => a !== '').map(a => <li key={a} style={{ fontFamily: 'monospace', fontSize: 12 }}>{a}</li>)}</ul>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add routes/access.js test/routes/access.test.js components/AccessList.js
git commit -m "feat: cds-auth access API + AccessList component"
```

---

## Task 12: Auth Sub-page + Docs Pages

**Files:**
- Create: `pages/components/[name]/auth.js`
- Create: `pages/docs/cds-auth.js`
- Create: `pages/docs/elasticsearch.js`

- [ ] **Step 1: Create pages/components/[name]/auth.js**

```jsx
// pages/components/[name]/auth.js
// Shows the cds-auth allowlist (jomax editable, cert/awsiam read-only)
// and links to the cds-auth docs page.
import { useRouter } from 'next/router';
import { useState } from 'react';
import Layout from '../../../components/Layout.js';
import ComponentNav from '../../../components/ComponentNav.js';
import AccessList from '../../../components/AccessList.js';

export default function AuthPage() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState(router.query.env || 'dev');

  function handleEnvChange(e) {
    setEnv(e);
    router.replace({ query: { ...router.query, env: e } });
  }

  return (
    <Layout env={env} onEnvChange={handleEnvChange}>
      <div style={{ padding: 24, maxWidth: 900, margin: '0 auto' }}>
        <a href="/" style={{ fontSize: 12, color: '#666' }}>← All components</a>
        <h1 style={{ margin: '8px 0 16px' }}>{name}</h1>
        {name && <ComponentNav name={name} env={env} />}

        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline', marginBottom: 16 }}>
          <h2 style={{ margin: 0 }}>Access Control</h2>
          <a href="/docs/cds-auth" target="_blank" style={{ fontSize: 13, color: '#0070d2' }}>
            How cds-auth works →
          </a>
        </div>

        <p style={{ color: '#555', fontSize: 13, marginBottom: 24 }}>
          Jomax usernames in the allowlist can deploy and modify this component.
          Cert and AWS IAM entries are managed via Switchboard directly or via the CDS CLI.
        </p>

        {name && <AccessList componentName={name} env={env} />}
      </div>
    </Layout>
  );
}
```

- [ ] **Step 2: Create pages/docs/cds-auth.js**

```jsx
// pages/docs/cds-auth.js
// Static explainer page for cds-auth. Linked from the Auth sub-page.
import Layout from '../../components/Layout.js';

export default function CdsAuthDocs() {
  return (
    <Layout>
      <div style={{ padding: 24, maxWidth: 760, margin: '0 auto', lineHeight: 1.7 }}>
        <a href="javascript:history.back()" style={{ fontSize: 12, color: '#666' }}>← Back</a>
        <h1>How cds-auth works</h1>
        <p>
          CDS uses a per-component allowlist stored in the <code>cds-auth</code> Switchboard app.
          Each component has a key matching its name (e.g. <code>bamMediaManagerEsm</code>) with a
          value like:
        </p>
        <pre style={{ background: '#f6f8fa', padding: 16, borderRadius: 4, overflow: 'auto' }}>{`{
  "jomax": ["jroig", "tbogart"],
  "cert": [""],
  "awsiam": [""]
}`}</pre>
        <h2>Auth types</h2>
        <dl>
          <dt><strong>jomax</strong></dt>
          <dd>GoDaddy Jomax SSO usernames. Engineers use jomax auth when deploying via the CDS CLI or this admin tool.</dd>
          <dt><strong>cert</strong></dt>
          <dd>Client certificate identities (e.g. service accounts). Used by automated pipelines. Edit via Switchboard directly or the CDS CLI.</dd>
          <dt><strong>awsiam</strong></dt>
          <dd>AWS IAM ARNs. Used by CI/CD pipelines running on AWS. Edit via Switchboard directly or the CDS CLI.</dd>
        </dl>
        <h2>Editing the allowlist</h2>
        <ul>
          <li>Jomax users — edit via the Auth tab in this tool</li>
          <li>Cert / AWS IAM — edit via Switchboard directly, or use the CDS CLI: <code>cds access add --cert &lt;identity&gt;</code></li>
        </ul>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 3: Create pages/docs/elasticsearch.js**

```jsx
// pages/docs/elasticsearch.js
// Static reference page explaining the two Elasticsearch clusters and their key fields.
// Linked from the Logs sub-page. Gives engineers a single place to understand
// what data is available and how it's indexed, without needing to poke at the clusters.
import Layout from '../../components/Layout.js';

export default function ElasticsearchDocs() {
  const logsExample = JSON.stringify({
    '@timestamp': '2026-01-15T14:23:11.042Z',
    'event': { 'type': 'bamMediaManagerEsm' },
    'log': { 'level': 'error' },
    'message': 'Failed to load module: network timeout after 3000ms',
    'error': { 'stack_trace': 'Error: network timeout\n  at fetch (/cds-loader/src/loader.js:142:11)' },
    'service': { 'name': 'websites-godaddy' },
    'http': { 'request': { 'referrer': 'https://www.godaddy.com/domains/domain-name-search' } },
    'client': { 'user': { 'id': 'shopper-82a4f1c3' } }
  }, null, 2);

  const rumExample = JSON.stringify({
    '@timestamp': '2026-01-15T14:23:15.890Z',
    'customProps': { 'media_library': { 'componentType': 'bamMediaManagerEsm' } },
    'lcp': 1842.5,
    'cls': 0.04,
    'inp': 112.0
  }, null, 2);

  const pre = { background: '#f6f8fa', padding: 16, borderRadius: 4, overflow: 'auto', fontSize: 13 };
  const dt = { marginTop: 12, fontWeight: 600 };
  const dd = { marginLeft: 16, marginTop: 2, color: '#555' };

  return (
    <Layout>
      <div style={{ padding: 24, maxWidth: 800, margin: '0 auto', lineHeight: 1.7 }}>
        <a href="javascript:history.back()" style={{ fontSize: 12, color: '#666' }}>← Back</a>
        <h1>Elasticsearch Data Sources</h1>
        <p>
          CDS Admin reads from two separate Elasticsearch clusters. Both are <strong>read-only</strong> from
          the app's perspective — data is written by <code>cds-loader</code> and the RUM poller, not by this tool.
          API keys stay server-side; the browser only hits <code>/api/analytics/*</code> proxy routes.
        </p>

        <h2>Logs cluster — <code>.ds-cds-api-prod-logs-*</code></h2>
        <p>
          Captures every CDS API call and error. Written by <code>cds-loader</code> when components are
          loaded on consumer pages. This is the data behind the <strong>Logs sub-page</strong> charts,
          raw log table, and errors-by-shopper breakdown.
        </p>
        <h3>Key fields</h3>
        <dl>
          <dt style={dt}><code>event.type</code></dt>
          <dd style={dd}>Component name (e.g. <code>bamMediaManagerEsm</code>). <strong>Primary filter for all log queries.</strong></dd>
          <dt style={dt}><code>log.level</code></dt>
          <dd style={dd}><code>error</code>, <code>warn</code>, or <code>info</code>. The Logs page level-filter uses this field.</dd>
          <dt style={dt}><code>message</code></dt>
          <dd style={dd}>Log message body. For request-volume queries, the <code>message.keyword</code> field is regex-matched for <code>componentTypes=</code>.</dd>
          <dt style={dt}><code>error.stack_trace</code></dt>
          <dd style={dd}>Stack trace. Only present on <code>log.level: error</code> entries. Shown in the collapsible expander in the raw log table.</dd>
          <dt style={dt}><code>service.name</code></dt>
          <dd style={dd}>The consuming app that loaded the component (e.g. <code>websites-godaddy</code>). Shown as "App" in the log table.</dd>
          <dt style={dt}><code>http.request.referrer</code></dt>
          <dd style={dd}>The page URL where the component was loaded. Shown as "Page URL" in the log table.</dd>
          <dt style={dt}><code>client.user.id</code></dt>
          <dd style={dd}>Shopper ID. Used for the errors-by-shopper aggregation (<code>terms</code> agg, top 10 by count).</dd>
        </dl>
        <h3>Example document</h3>
        <pre style={pre}>{logsExample}</pre>

        <h2 style={{ marginTop: 32 }}>RUM cluster — <code>sns-traffic-c1-prod*</code></h2>
        <p>
          Captures Core Web Vitals measurements from real users. Written by the RUM poller that runs
          alongside each component on the page. This is the data behind the <strong>RUM sub-page</strong>
          charts and the Web Vitals sparklines on the main dashboard.
        </p>
        <h3>Key fields</h3>
        <dl>
          <dt style={dt}><code>customProps.media_library.componentType</code></dt>
          <dd style={dd}>Component name. <strong>Primary filter for all RUM queries.</strong></dd>
          <dt style={dt}><code>lcp</code></dt>
          <dd style={dd}>Largest Contentful Paint in ms. Target: &lt; 2500ms. Shown as LCP p75.</dd>
          <dt style={dt}><code>cls</code></dt>
          <dd style={dd}>Cumulative Layout Shift (unitless). Target: &lt; 0.1. Shown as CLS p75.</dd>
          <dt style={dt}><code>inp</code></dt>
          <dd style={dd}>Interaction to Next Paint in ms. Target: &lt; 200ms. Shown as INP p75.</dd>
        </dl>
        <h3>Example document</h3>
        <pre style={pre}>{rumExample}</pre>

        <h2 style={{ marginTop: 32 }}>Kibana access</h2>
        <p>
          Both clusters have Kibana. The <code>kibanaDiscover</code> config in <code>gasket.js</code>
          per environment provides the <code>logsUrl</code> and <code>logsDataViewId</code> used to build
          Kibana Discover deep-links on the Logs sub-page. Dashboard deep-links use <code>kibanaDashboards.logs</code>
          and <code>kibanaDashboards.rum</code>.
        </p>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 4: Verify in browser** — navigate to `/components/{name}/auth`, add/remove a user, click "How cds-auth works →"; navigate to `/docs/elasticsearch`, confirm both clusters, fields, and example docs render.

- [ ] **Step 5: Commit**

```bash
git add pages/components/ pages/docs/
git commit -m "feat: auth sub-page + cds-auth + elasticsearch docs pages"
```

---

## Task 13: Sync API + Sync Sub-page

**Files:**
- Add to `routes/components.js`: POST `:name/sync`
- Create: `components/SyncPanel.js`
- Create: `pages/components/[name]/sync.js`

- [ ] **Step 1: Add test**

Add to `test/routes/components.test.js`:

```js
test('POST /api/components/:name/sync copies manifest and checks allowlist on toEnv', async () => {
  const { getCdsAuthClient, getCdsClient } = await import('../../clients/switchboard.js');
  const authGet = jest.fn().mockResolvedValue({ jomax: ['jroig'] });
  getCdsAuthClient.mockReturnValue({ get: authGet });
  const srcManifest = { engine: 'esm', js: 'abc.es.js', dependencies: {} };
  const srcGet = jest.fn().mockResolvedValue(srcManifest);
  const dstPut = jest.fn().mockResolvedValue();
  getCdsClient
    .mockImplementationOnce(() => ({ get: srcGet }))
    .mockImplementationOnce(() => ({ put: dstPut }));

  const res = await request(app)
    .post('/api/components/componentA/sync')
    .send({ fromEnv: 'prod', toEnv: 'test' });

  expect(res.status).toBe(200);
  expect(getCdsAuthClient).toHaveBeenCalledWith('test');
  expect(dstPut).toHaveBeenCalledWith('registry.componentA', srcManifest);
});

test('POST /api/components/:name/sync returns 403 when not allowlisted on toEnv', async () => {
  const { getCdsAuthClient } = await import('../../clients/switchboard.js');
  getCdsAuthClient.mockReturnValue({ get: jest.fn().mockResolvedValue({ jomax: ['alice'] }) });

  const res = await request(app)
    .post('/api/components/componentA/sync')
    .send({ fromEnv: 'dev', toEnv: 'prod' });

  expect(res.status).toBe(403);
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Add sync route to routes/components.js**

```js
router.post('/api/components/:name/sync', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const { fromEnv, toEnv } = req.body;
    // Allowlist check on DESTINATION env — prevents a dev user from pushing to prod.
    await assertCanAccess(name, req.user.accountName, toEnv);
    const manifest = await getCdsClient(fromEnv).get(`registry.${name}`);
    await getCdsClient(toEnv).put(`registry.${name}`, manifest);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create components/SyncPanel.js**

```jsx
// components/SyncPanel.js
import { useState } from 'react';

const ENVS = ['dev', 'test', 'prod'];

export default function SyncPanel({ componentName }) {
  const [status, setStatus] = useState('');
  const [busy, setBusy] = useState(false);

  async function handleSync(fromEnv, toEnv) {
    if (!confirm(`Copy ${componentName} manifest from ${fromEnv} → ${toEnv}?`)) return;
    setBusy(true);
    setStatus('');
    const res = await fetch(`/api/components/${componentName}/sync`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ fromEnv, toEnv })
    });
    setBusy(false);
    setStatus(res.ok ? `Synced ${fromEnv} → ${toEnv}` : 'Sync failed — check your access on the destination env');
  }

  return (
    <div>
      {status && <p style={{ color: status.includes('failed') ? 'red' : 'green', marginBottom: 16 }}>{status}</p>}
      <table style={{ borderCollapse: 'collapse' }}>
        <thead>
          <tr style={{ borderBottom: '2px solid #eee', fontSize: 12, color: '#666' }}>
            <th style={{ padding: '6px 16px', textAlign: 'left' }}>From</th>
            <th style={{ padding: '6px 16px', textAlign: 'left' }}>To</th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          {ENVS.flatMap(from =>
            ENVS.filter(to => to !== from).map(to => (
              <tr key={`${from}-${to}`} style={{ borderBottom: '1px solid #eee' }}>
                <td style={{ padding: '10px 16px' }}>{from}</td>
                <td style={{ padding: '10px 16px' }}>{to}</td>
                <td style={{ padding: '10px 16px' }}>
                  <button onClick={() => handleSync(from, to)} disabled={busy} style={{ fontSize: 12 }}>
                    Copy {from} → {to}
                  </button>
                </td>
              </tr>
            ))
          )}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 6: Create pages/components/[name]/sync.js**

```jsx
// pages/components/[name]/sync.js
import { useRouter } from 'next/router';
import { useState } from 'react';
import Layout from '../../../components/Layout.js';
import ComponentNav from '../../../components/ComponentNav.js';
import SyncPanel from '../../../components/SyncPanel.js';

export default function SyncPage() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState(router.query.env || 'dev');

  function handleEnvChange(e) {
    setEnv(e);
    router.replace({ query: { ...router.query, env: e } });
  }

  return (
    <Layout env={env} onEnvChange={handleEnvChange}>
      <div style={{ padding: 24, maxWidth: 900, margin: '0 auto' }}>
        <a href="/" style={{ fontSize: 12, color: '#666' }}>← All components</a>
        <h1 style={{ margin: '8px 0 16px' }}>{name}</h1>
        {name && <ComponentNav name={name} env={env} />}
        <h2>Environment Sync</h2>
        <p style={{ color: '#555', fontSize: 13, marginBottom: 24 }}>
          Copy the manifest from one environment to another. You must be allowlisted on the destination environment.
        </p>
        {name && <SyncPanel componentName={name} />}
      </div>
    </Layout>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add routes/components.js components/SyncPanel.js pages/components/
git commit -m "feat: sync API + sync sub-page"
```

---

## Task 14: Elasticsearch Client (with Log Level Filter)

**Files:**
- Create: `clients/elasticsearch.js`
- Create: `test/clients/elasticsearch.test.js`

`queryErrors` now accepts an optional `level` param (`'error'` | `'warn'` | `'info'` | `'all'`). When `level === 'all'`, the log.level term filter is omitted.

- [ ] **Step 1: Write failing tests**

```js
// test/clients/elasticsearch.test.js
import { jest } from '@jest/globals';

const mockLogsSearch = jest.fn().mockResolvedValue({ hits: { total: { value: 5 } }, aggregations: {} });
const mockRumSearch = jest.fn().mockResolvedValue({ hits: { total: { value: 0 } }, aggregations: {} });

jest.unstable_mockModule('@elastic/elasticsearch', () => ({
  Client: jest.fn().mockImplementation(({ node }) => ({
    search: node.includes('rum') ? mockRumSearch : mockLogsSearch
  }))
}));

process.env.ELASTICSEARCH_LOGS_URL = 'https://logs.example.com';
process.env.ELASTICSEARCH_LOGS_API_KEY = 'bG9nczp0ZXN0';
process.env.ELASTICSEARCH_RUM_URL = 'https://rum.example.com';
process.env.ELASTICSEARCH_RUM_API_KEY = 'cnVtOnRlc3Q=';

const { getLogsClient, getRumClient, queryErrors, queryRequestVolume, queryWebVitals, queryRawLogs, queryErrorsByUser } = await import('../../clients/elasticsearch.js');

test('getLogsClient returns singleton', () => {
  expect(getLogsClient()).toBe(getLogsClient());
});

test('getRumClient is distinct from logs client', () => {
  expect(getLogsClient()).not.toBe(getRumClient());
});

test('queryErrors with level=error filters by log.level', async () => {
  await queryErrors('myComponent', 'now-1d', 'now', 'error');
  expect(mockLogsSearch).toHaveBeenCalledWith(expect.objectContaining({
    index: '.ds-cds-api-prod-logs-*',
    body: expect.objectContaining({
      query: expect.objectContaining({
        bool: expect.objectContaining({
          must: expect.arrayContaining([{ term: { 'log.level': 'error' } }])
        })
      })
    })
  }));
});

test('queryErrors with level=all omits log.level filter', async () => {
  mockLogsSearch.mockClear();
  await queryErrors('myComponent', 'now-1d', 'now', 'all');
  const body = mockLogsSearch.mock.calls[0][0].body;
  const must = body.query.bool.must;
  expect(must.some(c => c.term?.['log.level'])).toBe(false);
});

test('queryWebVitals calls RUM cluster', async () => {
  await queryWebVitals('myComponent', 'now-1d', 'now');
  expect(mockRumSearch).toHaveBeenCalledWith(expect.objectContaining({
    index: 'sns-traffic-c1-prod*'
  }));
});

test('queryRawLogs returns size 100 hits sorted desc', async () => {
  mockLogsSearch.mockResolvedValueOnce({ hits: { hits: [] } });
  await queryRawLogs('myComponent', 'now-1d', 'now', 'all', 100);
  expect(mockLogsSearch).toHaveBeenCalledWith(expect.objectContaining({
    body: expect.objectContaining({
      size: 100,
      sort: [{ '@timestamp': { order: 'desc' } }]
    })
  }));
});

test('queryRawLogs with level=error adds log.level filter', async () => {
  mockLogsSearch.mockResolvedValueOnce({ hits: { hits: [] } });
  await queryRawLogs('myComponent', 'now-1d', 'now', 'error');
  const body = mockLogsSearch.mock.calls[mockLogsSearch.mock.calls.length - 1][0].body;
  expect(body.query.bool.must).toEqual(expect.arrayContaining([{ term: { 'log.level': 'error' } }]));
});

test('queryErrorsByUser returns terms agg on client.user.id', async () => {
  mockLogsSearch.mockResolvedValueOnce({
    hits: { total: { value: 50 } },
    aggregations: { by_user: { buckets: [{ key: 'shopper-123', doc_count: 30 }] } }
  });
  await queryErrorsByUser('myComponent', 'now-1d', 'now');
  expect(mockLogsSearch).toHaveBeenCalledWith(expect.objectContaining({
    body: expect.objectContaining({
      aggs: { by_user: { terms: expect.objectContaining({ field: 'client.user.id', size: 10 }) } }
    })
  }));
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create clients/elasticsearch.js**

```js
// clients/elasticsearch.js
import { Client } from '@elastic/elasticsearch';

let logsClient;
let rumClient;

export function getLogsClient() {
  if (!logsClient) {
    logsClient = new Client({
      node: process.env.ELASTICSEARCH_LOGS_URL,
      auth: { apiKey: process.env.ELASTICSEARCH_LOGS_API_KEY }
    });
  }
  return logsClient;
}

export function getRumClient() {
  if (!rumClient) {
    rumClient = new Client({
      node: process.env.ELASTICSEARCH_RUM_URL,
      auth: { apiKey: process.env.ELASTICSEARCH_RUM_API_KEY }
    });
  }
  return rumClient;
}

// level: 'error' | 'warn' | 'info' | 'all' — 'all' omits the log.level filter
export async function queryErrors(componentName, from, to, level = 'error') {
  const must = [
    { term: { 'event.type': componentName } },
    { range: { '@timestamp': { gte: from, lte: to } } }
  ];
  if (level !== 'all') must.unshift({ term: { 'log.level': level } });

  return getLogsClient().search({
    index: '.ds-cds-api-prod-logs-*',
    body: {
      size: 0,
      query: { bool: { must } },
      aggs: {
        over_time: { date_histogram: { field: '@timestamp', fixed_interval: '30m', min_doc_count: 0 } },
        total: { value_count: { field: '@timestamp' } }
      }
    }
  });
}

export async function queryRequestVolume(componentName, from, to) {
  const escaped = componentName.replace(/[.?+*|{}[\]()"\\]/g, '\\$&');
  return getLogsClient().search({
    index: '.ds-cds-api-prod-logs-*',
    body: {
      size: 0,
      query: {
        bool: {
          must: [
            { term: { 'log.level': 'info' } },
            { regexp: { 'message.keyword': `.*componentTypes=([^&]*,)?${escaped}(,[^& ]*)?.*` } },
            { range: { '@timestamp': { gte: from, lte: to } } }
          ]
        }
      },
      aggs: { over_time: { date_histogram: { field: '@timestamp', fixed_interval: '1h', min_doc_count: 0 } } }
    }
  });
}

export async function queryWebVitals(componentName, from, to) {
  return getRumClient().search({
    index: 'sns-traffic-c1-prod*',
    body: {
      size: 0,
      query: {
        bool: {
          must: [
            { term: { 'customProps.media_library.componentType': componentName } },
            { range: { '@timestamp': { gte: from, lte: to } } }
          ]
        }
      },
      aggs: {
        over_time: {
          date_histogram: { field: '@timestamp', fixed_interval: '1h', min_doc_count: 0 },
          aggs: {
            lcp_p75: { percentiles: { field: 'lcp', percents: [75] } },
            cls_p75: { percentiles: { field: 'cls', percents: [75] } },
            inp_p75: { percentiles: { field: 'inp', percents: [75] } }
          }
        }
      }
    }
  });
}

// Returns the last `size` raw log entries for a component, newest first.
// Fields: @timestamp, log.level, message, error.stack_trace, service.name (consuming app),
// http.request.referrer (page URL), client.user.id (shopperId).
// level='all' omits the log.level filter.
export async function queryRawLogs(componentName, from, to, level = 'all', size = 100) {
  const must = [
    { term: { 'event.type': componentName } },
    { range: { '@timestamp': { gte: from, lte: to } } }
  ];
  if (level !== 'all') must.push({ term: { 'log.level': level } });

  return getLogsClient().search({
    index: '.ds-cds-api-prod-logs-*',
    body: {
      size,
      _source: ['@timestamp', 'log.level', 'message', 'error.stack_trace', 'service.name', 'http.request.referrer', 'client.user.id'],
      query: { bool: { must } },
      sort: [{ '@timestamp': { order: 'desc' } }]
    }
  });
}

// Returns top shoppers (client.user.id) by error count for a component.
// Useful for spotting if a small number of shoppers account for disproportionate errors.
export async function queryErrorsByUser(componentName, from, to) {
  return getLogsClient().search({
    index: '.ds-cds-api-prod-logs-*',
    body: {
      size: 0,
      query: {
        bool: {
          must: [
            { term: { 'event.type': componentName } },
            { term: { 'log.level': 'error' } },
            { range: { '@timestamp': { gte: from, lte: to } } }
          ]
        }
      },
      aggs: {
        by_user: {
          terms: { field: 'client.user.id', size: 10, order: { _count: 'desc' } }
        }
      }
    }
  });
}
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Commit**

```bash
git add clients/elasticsearch.js test/clients/elasticsearch.test.js
git commit -m "feat: elasticsearch client with log level filter support"
```

---

## Task 15: Analytics API Routes (with Level Filter)

**Files:**
- Create: `routes/analytics.js`
- Create: `test/routes/analytics.test.js`
- Create: `components/charts/ErrorsOverTime.js`
- Create: `components/charts/RequestVolume.js`
- Create: `components/charts/WebVitals.js`
- Create: `components/charts/MiniChart.js`
- Create: `pages/analytics.js`

The `/api/analytics/errors` route accepts an optional `level` query param (default `'error'`). All chart components are created here since they're shared across pages.

- [ ] **Step 1: Write failing tests**

```js
// test/routes/analytics.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockQueryErrors = jest.fn().mockResolvedValue({
  hits: { total: { value: 12 } },
  aggregations: { over_time: { buckets: [] } }
});

const mockQueryRawLogs = jest.fn().mockResolvedValue({ hits: { hits: [] } });
const mockQueryErrorsByUser = jest.fn().mockResolvedValue({ aggregations: { by_user: { buckets: [] } } });

jest.unstable_mockModule('../../clients/elasticsearch.js', () => ({
  queryErrors: mockQueryErrors,
  queryRequestVolume: jest.fn().mockResolvedValue({ aggregations: { over_time: { buckets: [] } } }),
  queryWebVitals: jest.fn().mockResolvedValue({ aggregations: { over_time: { buckets: [] } } }),
  queryRawLogs: mockQueryRawLogs,
  queryErrorsByUser: mockQueryErrorsByUser
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/analytics.js');
const app = express();
app.use(router);

test('GET /api/analytics/errors passes level=error by default', async () => {
  await request(app).get('/api/analytics/errors?component=myComp&from=now-1d&to=now');
  expect(mockQueryErrors).toHaveBeenCalledWith('myComp', 'now-1d', 'now', 'error');
});

test('GET /api/analytics/errors passes level=all when specified', async () => {
  await request(app).get('/api/analytics/errors?component=myComp&from=now-1d&to=now&level=all');
  expect(mockQueryErrors).toHaveBeenCalledWith('myComp', 'now-1d', 'now', 'all');
});

test('GET /api/analytics/errors returns total and buckets', async () => {
  const res = await request(app).get('/api/analytics/errors?component=myComp');
  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('total');
  expect(res.body).toHaveProperty('buckets');
});

test('GET /api/analytics/volume returns buckets', async () => {
  const res = await request(app).get('/api/analytics/volume?component=myComp');
  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('buckets');
});

test('GET /api/analytics/vitals returns buckets', async () => {
  const res = await request(app).get('/api/analytics/vitals?component=myComp');
  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('buckets');
});

test('GET /api/analytics/logs returns entries array with expected fields', async () => {
  mockQueryRawLogs.mockResolvedValueOnce({
    hits: { hits: [{ _source: { '@timestamp': '2026-01-01T00:00:00Z', 'log.level': 'error', message: 'failed', 'service.name': 'myApp', 'http.request.referrer': 'https://example.com', 'client.user.id': 'shopper-abc' } }] }
  });
  const res = await request(app).get('/api/analytics/logs?component=myComp');
  expect(res.status).toBe(200);
  expect(res.body.entries).toHaveLength(1);
  expect(res.body.entries[0]).toMatchObject({
    timestamp: '2026-01-01T00:00:00Z',
    level: 'error',
    message: 'failed',
    app: 'myApp',
    shopperId: 'shopper-abc'
  });
});

test('GET /api/analytics/errors-by-user returns shoppers array', async () => {
  mockQueryErrorsByUser.mockResolvedValueOnce({
    aggregations: { by_user: { buckets: [{ key: 'shopper-123', doc_count: 42 }] } }
  });
  const res = await request(app).get('/api/analytics/errors-by-user?component=myComp');
  expect(res.status).toBe(200);
  expect(res.body.shoppers).toEqual([{ shopperId: 'shopper-123', count: 42 }]);
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create routes/analytics.js**

```js
// routes/analytics.js
import express from 'express';
import { queryErrors, queryRequestVolume, queryWebVitals, queryRawLogs, queryErrorsByUser } from '../clients/elasticsearch.js';
import { requireJomax } from '../middleware/require-jomax.js';

const router = express.Router();

router.get('/api/analytics/errors', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now', level = 'error' } = req.query;
    const result = await queryErrors(component, from, to, level);
    res.json({
      total: result.hits.total.value,
      buckets: result.aggregations.over_time.buckets.map(b => ({ time: b.key_as_string, count: b.doc_count }))
    });
  } catch (err) { next(err); }
});

router.get('/api/analytics/volume', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now' } = req.query;
    const result = await queryRequestVolume(component, from, to);
    res.json({ buckets: result.aggregations.over_time.buckets.map(b => ({ time: b.key_as_string, count: b.doc_count })) });
  } catch (err) { next(err); }
});

router.get('/api/analytics/vitals', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now' } = req.query;
    const result = await queryWebVitals(component, from, to);
    res.json({
      buckets: result.aggregations.over_time.buckets.map(b => ({
        time: b.key_as_string,
        lcp_p75: b.lcp_p75?.values?.['75.0'] ?? null,
        cls_p75: b.cls_p75?.values?.['75.0'] ?? null,
        inp_p75: b.inp_p75?.values?.['75.0'] ?? null
      }))
    });
  } catch (err) { next(err); }
});

// Returns last 100 raw log entries. level param (default 'all') filters by log.level.
// Response shape: { entries: [{ timestamp, level, message, stackTrace, app, pageUrl, shopperId }] }
router.get('/api/analytics/logs', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now', level = 'all' } = req.query;
    const result = await queryRawLogs(component, from, to, level, 100);
    const entries = result.hits.hits.map(h => ({
      timestamp: h._source['@timestamp'],
      level: h._source['log.level'],
      message: h._source.message,
      stackTrace: h._source['error.stack_trace'] ?? null,
      app: h._source['service.name'] ?? null,
      pageUrl: h._source['http.request.referrer'] ?? null,
      shopperId: h._source['client.user.id'] ?? null
    }));
    res.json({ entries });
  } catch (err) { next(err); }
});

// Returns top shoppers (client.user.id) by error count.
// Response shape: { shoppers: [{ shopperId, count }] }
router.get('/api/analytics/errors-by-user', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now' } = req.query;
    const result = await queryErrorsByUser(component, from, to);
    const shoppers = result.aggregations.by_user.buckets.map(b => ({
      shopperId: b.key,
      count: b.doc_count
    }));
    res.json({ shoppers });
  } catch (err) { next(err); }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create chart components**

```jsx
// components/charts/ErrorsOverTime.js
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, ReferenceLine } from 'recharts';

export default function ErrorsOverTime({ data, deployedAt }) {
  return (
    <ResponsiveContainer width="100%" height={200}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tick={{ fontSize: 11 }} />
        <YAxis allowDecimals={false} />
        <Tooltip />
        {deployedAt && <ReferenceLine x={deployedAt} stroke="#f56565" strokeDasharray="4 2" label={{ value: 'Deploy', position: 'top', fontSize: 11 }} />}
        <Line type="monotone" dataKey="count" stroke="#e53e3e" dot={false} />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```jsx
// components/charts/RequestVolume.js
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, ReferenceLine } from 'recharts';

export default function RequestVolume({ data, deployedAt }) {
  return (
    <ResponsiveContainer width="100%" height={200}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tick={{ fontSize: 11 }} />
        <YAxis allowDecimals={false} />
        <Tooltip />
        {deployedAt && <ReferenceLine x={deployedAt} stroke="#f56565" strokeDasharray="4 2" label={{ value: 'Deploy', position: 'top', fontSize: 11 }} />}
        <Line type="monotone" dataKey="count" stroke="#38a169" dot={false} />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```jsx
// components/charts/WebVitals.js
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer, ReferenceLine } from 'recharts';

export default function WebVitals({ data, deployedAt }) {
  return (
    <ResponsiveContainer width="100%" height={250}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tick={{ fontSize: 11 }} />
        <YAxis />
        <Tooltip />
        <Legend />
        {deployedAt && <ReferenceLine x={deployedAt} stroke="#f56565" strokeDasharray="4 2" label={{ value: 'Deploy', position: 'top', fontSize: 11 }} />}
        <Line type="monotone" dataKey="lcp_p75" name="LCP p75 (ms)" stroke="#3182ce" dot={false} />
        <Line type="monotone" dataKey="inp_p75" name="INP p75 (ms)" stroke="#805ad5" dot={false} />
        <Line type="monotone" dataKey="cls_p75" name="CLS p75" stroke="#d69e2e" dot={false} />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```jsx
// components/charts/MiniChart.js
import { LineChart, Line, YAxis, ResponsiveContainer, Tooltip } from 'recharts';

export default function MiniChart({ data, dataKey, color, label, unit = '' }) {
  const latest = data?.length ? data[data.length - 1][dataKey] : null;
  return (
    <div style={{ border: '1px solid #eee', borderRadius: 4, padding: 12 }}>
      <div style={{ fontSize: 11, color: '#666', textTransform: 'uppercase', letterSpacing: 0.5, marginBottom: 2 }}>{label}</div>
      <div style={{ fontSize: 22, fontWeight: 'bold', marginBottom: 4 }}>
        {latest != null ? `${Math.round(latest)}${unit}` : '—'}
      </div>
      <ResponsiveContainer width="100%" height={60}>
        <LineChart data={data}>
          <YAxis hide domain={['auto', 'auto']} />
          <Tooltip />
          <Line type="monotone" dataKey={dataKey} stroke={color} strokeWidth={2} dot={false} />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

- [ ] **Step 6: Create pages/analytics.js** (standalone analytics page)

```jsx
// pages/analytics.js
import { useState, useEffect } from 'react';
import Layout from '../components/Layout.js';
import ErrorsOverTime from '../components/charts/ErrorsOverTime.js';
import RequestVolume from '../components/charts/RequestVolume.js';
import WebVitals from '../components/charts/WebVitals.js';

const TIME_RANGES = [
  { label: '1h', value: 'now-1h' },
  { label: '6h', value: 'now-6h' },
  { label: '24h', value: 'now-1d' },
  { label: '7d', value: 'now-7d' }
];

function kibanaLink(baseUrl, kql) {
  if (!baseUrl) return null;
  return `${baseUrl}&_a=${encodeURIComponent(`(query:(language:kuery,query:'${kql}'))`)}`;
}

export default function Analytics() {
  const [components, setComponents] = useState([]);
  const [component, setComponent] = useState('');
  const [range, setRange] = useState('now-1d');
  const [errors, setErrors] = useState({ total: 0, buckets: [] });
  const [volume, setVolume] = useState({ buckets: [] });
  const [vitals, setVitals] = useState({ buckets: [] });
  const [loading, setLoading] = useState(false);
  const [kibanaDashboards, setKibanaDashboards] = useState({});

  useEffect(() => {
    fetch('/api/config').then(r => r.json()).then(d => setKibanaDashboards(d.kibanaDashboards ?? {}));
    fetch('/api/components?env=prod').then(r => r.json())
      .then(items => {
        const names = items.map(c => c.name);
        setComponents(names);
        if (names.length) setComponent(names[0]);
      });
  }, []);

  useEffect(() => {
    if (!component) return;
    setLoading(true);
    const p = `component=${component}&from=${range}&to=now`;
    Promise.all([
      fetch(`/api/analytics/errors?${p}`).then(r => r.json()),
      fetch(`/api/analytics/volume?${p}`).then(r => r.json()),
      fetch(`/api/analytics/vitals?${p}`).then(r => r.json())
    ]).then(([e, v, w]) => { setErrors(e); setVolume(v); setVitals(w); setLoading(false); });
  }, [component, range]);

  return (
    <Layout>
      <div style={{ padding: 24 }}>
        <h1>Analytics</h1>
        <div style={{ display: 'flex', gap: 16, marginBottom: 24, alignItems: 'center', flexWrap: 'wrap' }}>
          <select value={component} onChange={e => setComponent(e.target.value)}>
            {components.map(c => <option key={c} value={c}>{c}</option>)}
          </select>
          <div style={{ display: 'flex', gap: 8 }}>
            {TIME_RANGES.map(r => (
              <button key={r.value} onClick={() => setRange(r.value)}
                style={{ fontWeight: range === r.value ? 'bold' : 'normal' }}>{r.label}</button>
            ))}
          </div>
          {loading && <span style={{ color: '#999', fontSize: 13 }}>Loading…</span>}
        </div>
        <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: 24 }}>
          <div>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline' }}>
              <h3>Errors — {errors.total} total</h3>
              {kibanaDashboards.logs && <a href={kibanaLink(kibanaDashboards.logs, `event.type:"${component}"`)} target="_blank" rel="noreferrer" style={{ fontSize: 12, color: '#0070d2' }}>Open in Kibana →</a>}
            </div>
            <ErrorsOverTime data={errors.buckets} />
          </div>
          <div>
            <h3>Request volume</h3>
            <RequestVolume data={volume.buckets} />
          </div>
          <div style={{ gridColumn: '1 / -1' }}>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline' }}>
              <h3>Web Vitals (p75)</h3>
              {kibanaDashboards.rum && <a href={kibanaLink(kibanaDashboards.rum, `customProps.media_library.componentType:"${component}"`)} target="_blank" rel="noreferrer" style={{ fontSize: 12, color: '#0070d2' }}>Open in Kibana →</a>}
            </div>
            <WebVitals data={vitals.buckets} />
          </div>
        </div>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add routes/analytics.js test/routes/analytics.test.js components/charts/ pages/analytics.js
git commit -m "feat: analytics API with level filter + chart components + analytics page"
```

---

## Task 16: RUM Sub-page (with Deployment Markers)

**Files:**
- Create: `routes/rum.js`
- Create: `test/routes/rum.test.js`
- Create: `pages/components/[name]/rum.js`

The RUM sub-page shows full Web Vitals charts (LCP/CLS/INP p75) with a vertical `ReferenceLine` at `metaData.updatedAt` — the last time the manifest was deployed. This lets engineers visually see if a deploy changed performance.

- [ ] **Step 1: Write failing tests for routes/rum.js**

```js
// test/routes/rum.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockGet, put: mockPut, getHistory: jest.fn().mockResolvedValue([]) })),
  getCdsAuthClient: jest.fn(() => ({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/rum.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components/:name/rum returns RUM config', async () => {
  const rumConfig = { pollerInterval: 100, pollerTimeout: 60000, targetDomSelector: '[data-testid="header"]' };
  mockGet.mockResolvedValue(rumConfig);
  const res = await request(app).get('/api/components/componentA/rum?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual(rumConfig);
});

test('PUT /api/components/:name/rum updates RUM config', async () => {
  mockPut.mockResolvedValue();
  const rumConfig = { pollerInterval: 200, pollerTimeout: 30000 };
  const res = await request(app)
    .put('/api/components/componentA/rum?env=dev')
    .send(rumConfig);
  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('registry.componentA.rum', rumConfig);
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create routes/rum.js**

```js
// routes/rum.js
import express from 'express';
import { getCdsClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/components/:name/rum', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const value = await getCdsClient(env).get(`registry.${req.params.name}.rum`);
    if (value == null) return res.status(404).json({ error: 'RUM not configured' });
    res.json(value);
  } catch (err) { next(err); }
});

router.put('/api/components/:name/rum', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.rum`, req.body);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.get('/api/components/:name/rum/history', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const history = await getCdsClient(env).getHistory(`registry.${req.params.name}.rum`);
    res.json(history ?? []);
  } catch (err) { next(err); }
});

router.post('/api/components/:name/rum/rollback', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.rum`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create pages/components/[name]/rum.js**

```jsx
// pages/components/[name]/rum.js
// Full RUM Web Vitals charts with a deployment marker at metaData.updatedAt.
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import Layout from '../../../components/Layout.js';
import ComponentNav from '../../../components/ComponentNav.js';
import JsonEditor from '../../../components/JsonEditor.js';
import WebVitals from '../../../components/charts/WebVitals.js';

const TIME_RANGES = [
  { label: '1h', value: 'now-1h' },
  { label: '6h', value: 'now-6h' },
  { label: '24h', value: 'now-1d' },
  { label: '7d', value: 'now-7d' }
];

export default function RumPage() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState(router.query.env || 'dev');
  const [range, setRange] = useState('now-1d');
  const [vitals, setVitals] = useState({ buckets: [] });
  const [deployedAt, setDeployedAt] = useState(null);
  const [rumConfig, setRumConfig] = useState(null);
  const [editorValue, setEditorValue] = useState('');
  const [editorValid, setEditorValid] = useState(true);
  const [saving, setSaving] = useState(false);
  const [kibanaDashboards, setKibanaDashboards] = useState({});

  useEffect(() => {
    fetch('/api/config').then(r => r.json()).then(d => setKibanaDashboards(d.kibanaDashboards ?? {}));
  }, []);

  useEffect(() => {
    if (!name) return;
    fetch(`/api/components/${name}?env=${env}`).then(r => r.json())
      .then(m => setDeployedAt(m?.metaData?.updatedAt ?? null));
    fetch(`/api/components/${name}/rum?env=${env}`).then(r => r.ok ? r.json() : null)
      .then(cfg => { if (cfg) { setRumConfig(cfg); setEditorValue(JSON.stringify(cfg, null, 2)); } });
  }, [name, env]);

  useEffect(() => {
    if (!name) return;
    fetch(`/api/analytics/vitals?component=${name}&from=${range}&to=now`)
      .then(r => r.json()).then(setVitals);
  }, [name, env, range]);

  async function saveRum() {
    setSaving(true);
    await fetch(`/api/components/${name}/rum?env=${env}`, {
      method: 'PUT', headers: { 'content-type': 'application/json' }, body: editorValue
    });
    setSaving(false);
  }

  const kibanaLink = kibanaDashboards.rum && name
    ? `${kibanaDashboards.rum}&_a=${encodeURIComponent(`(query:(language:kuery,query:'customProps.media_library.componentType:"${name}"'))`)}`
    : null;

  function handleEnvChange(e) {
    setEnv(e);
    router.replace({ query: { ...router.query, env: e } });
  }

  return (
    <Layout env={env} onEnvChange={handleEnvChange}>
      <div style={{ padding: 24, maxWidth: 1200, margin: '0 auto' }}>
        <a href="/" style={{ fontSize: 12, color: '#666' }}>← All components</a>
        <h1 style={{ margin: '8px 0 16px' }}>{name}</h1>
        {name && <ComponentNav name={name} env={env} />}

        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 }}>
          <div style={{ display: 'flex', gap: 8 }}>
            {TIME_RANGES.map(r => (
              <button key={r.value} onClick={() => setRange(r.value)}
                style={{ fontWeight: range === r.value ? 'bold' : 'normal' }}>{r.label}</button>
            ))}
          </div>
          {kibanaLink && <a href={kibanaLink} target="_blank" rel="noreferrer" style={{ fontSize: 13, color: '#0070d2' }}>Open in Kibana →</a>}
        </div>

        {deployedAt && (
          <p style={{ fontSize: 12, color: '#555', marginBottom: 12 }}>
            Red line = last deploy ({new Date(deployedAt).toLocaleString()})
          </p>
        )}
        <WebVitals data={vitals.buckets} deployedAt={deployedAt} />

        <h3 style={{ marginTop: 32 }}>RUM Poller Config</h3>
        {!rumConfig && <p style={{ color: '#999' }}>Not configured yet.</p>}
        <JsonEditor value={editorValue} onChange={v => setEditorValue(v ?? '')}
          onValidate={setEditorValid} height="200px" />
        <button onClick={saveRum} disabled={!editorValid || saving} style={{ marginTop: 8 }}>
          {saving ? 'Saving…' : 'Save RUM Config'}
        </button>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add routes/rum.js test/routes/rum.test.js pages/components/
git commit -m "feat: RUM sub-page with deployment markers + RUM config editor"
```

---

## Task 17: Logs Sub-page (Charts + Raw Log Table + Shopper Breakdown)

**Files:**
- Create: `pages/components/[name]/logs.js`

The Logs sub-page has four sections:
1. **Charts** — error rate over time + request volume, with level filter (error/warn/info/all), time range picker, and deployment marker
2. **Errors by shopper** — top shoppers by error count from `queryErrorsByUser` (identifies if specific shoppers account for disproportionate errors)
3. **Raw log entries** — last 100 entries from `queryRawLogs`: timestamp, level badge, message, consuming app (`service.name`), page URL (`http.request.referrer`), shopper ID (`client.user.id`)
4. **Kibana Discover link** — deep-links directly to Kibana Discover pre-filtered to this component using `kibanaDiscover.logsUrl` + `logsDataViewId` from `/api/config`

The `buildKibanaDiscoverUrl(kibanaDiscover, componentName, from, to)` helper builds the full Kibana Discover URL using the Rison-encoded `_g` and `_a` params matching the format from the prod cluster.

Log level filter persists per-component in localStorage (`cds-admin:logLevel:{name}`), default `error`. Level change immediately refetches both chart data and raw entries. Time range picker (1h/6h/24h/7d) applies to charts; raw entries always show last 100 within the selected range.

- [ ] **Step 1: Create pages/components/[name]/logs.js**

```jsx
// pages/components/[name]/logs.js
// Four sections: charts, errors-by-shopper, raw log table, Kibana Discover link.
// Level filter (error/warn/info/all) persists in localStorage per component.
// Deployment marker at metaData.updatedAt on charts.
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import Layout from '../../../components/Layout.js';
import ComponentNav from '../../../components/ComponentNav.js';
import ErrorsOverTime from '../../../components/charts/ErrorsOverTime.js';
import RequestVolume from '../../../components/charts/RequestVolume.js';

const TIME_RANGES = [
  { label: '1h', value: 'now-1h' },
  { label: '6h', value: 'now-6h' },
  { label: '24h', value: 'now-1d' },
  { label: '7d', value: 'now-7d' }
];
const LOG_LEVELS = ['error', 'warn', 'info', 'all'];

// Builds the Kibana Discover deep-link URL. Matches the format used by the prod cluster:
// https://{logsUrl}/app/discover#/?_g=(time:...)&_a=(dataSource:(dataViewId:{id},...),query:...,sort:...)
function buildKibanaDiscoverUrl(kibanaDiscover, componentName, from, to) {
  if (!kibanaDiscover?.logsUrl || !kibanaDiscover?.logsDataViewId) return null;
  const { logsUrl, logsDataViewId } = kibanaDiscover;
  const g = `(filters:!(),query:(language:kuery,query:''),refreshInterval:(pause:!t,value:60000),time:(from:${from},to:${to}))`;
  const a = `(columns:!(),dataSource:(dataViewId:${logsDataViewId},type:dataView),filters:!(),hideChart:!f,interval:auto,query:(language:kuery,query:'event.type:${encodeURIComponent(`"${componentName}"`)}'),sort:!(!('@timestamp',desc)))`;
  return `${logsUrl}/app/discover#/?_g=${encodeURIComponent(g)}&_a=${encodeURIComponent(a)}`;
}

function getStoredLevel(componentName) {
  try { return localStorage.getItem(`cds-admin:logLevel:${componentName}`) ?? 'error'; }
  catch { return 'error'; }
}
function storeLevel(componentName, level) {
  try { localStorage.setItem(`cds-admin:logLevel:${componentName}`, level); } catch {}
}

const LEVEL_COLORS = { error: '#c53030', warn: '#d69e2e', info: '#2b6cb0', all: '#555' };

export default function LogsPage() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState(router.query.env || 'dev');
  const [range, setRange] = useState('now-1d');
  const [level, setLevel] = useState('error');
  const [errors, setErrors] = useState({ total: 0, buckets: [] });
  const [volume, setVolume] = useState({ buckets: [] });
  const [rawLogs, setRawLogs] = useState([]);
  const [shoppers, setShoppers] = useState([]);
  const [deployedAt, setDeployedAt] = useState(null);
  const [kibanaDiscover, setKibanaDiscover] = useState({});

  // Load config (kibanaDiscover) and level from localStorage once
  useEffect(() => {
    fetch('/api/config').then(r => r.json()).then(d => setKibanaDiscover(d.kibanaDiscover ?? {}));
  }, []);

  useEffect(() => {
    if (!name) return;
    setLevel(getStoredLevel(name));
  }, [name]);

  // Fetch last-deploy timestamp
  useEffect(() => {
    if (!name) return;
    fetch(`/api/components/${name}?env=${env}`).then(r => r.json())
      .then(m => setDeployedAt(m?.metaData?.updatedAt ?? null));
  }, [name, env]);

  // Fetch charts + raw logs + shoppers whenever name/range/level changes
  useEffect(() => {
    if (!name) return;
    const p = `component=${name}&from=${range}&to=now`;
    fetch(`/api/analytics/errors?${p}&level=${level}`).then(r => r.json()).then(setErrors);
    fetch(`/api/analytics/volume?${p}`).then(r => r.json()).then(setVolume);
    fetch(`/api/analytics/logs?${p}&level=${level}`).then(r => r.json()).then(d => setRawLogs(d.entries ?? []));
    fetch(`/api/analytics/errors-by-user?${p}`).then(r => r.json()).then(d => setShoppers(d.shoppers ?? []));
  }, [name, env, range, level]);

  function handleLevelChange(newLevel) {
    setLevel(newLevel);
    if (name) storeLevel(name, newLevel);
  }

  function handleEnvChange(e) {
    setEnv(e);
    router.replace({ query: { ...router.query, env: e } });
  }

  const kibanaLink = buildKibanaDiscoverUrl(kibanaDiscover, name, range, 'now');

  return (
    <Layout env={env} onEnvChange={handleEnvChange}>
      <div style={{ padding: 24, maxWidth: 1200, margin: '0 auto' }}>
        <a href="/" style={{ fontSize: 12, color: '#666' }}>← All components</a>
        <h1 style={{ margin: '8px 0 16px' }}>{name}</h1>
        {name && <ComponentNav name={name} env={env} />}

        {/* Controls row */}
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16, flexWrap: 'wrap', gap: 12 }}>
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <span style={{ fontSize: 13, color: '#555' }}>Level:</span>
            {LOG_LEVELS.map(l => (
              <button key={l} onClick={() => handleLevelChange(l)}
                style={{ fontWeight: level === l ? 'bold' : 'normal',
                  background: level === l ? '#eef' : undefined, borderRadius: 4, padding: '4px 10px',
                  color: level === l ? LEVEL_COLORS[l] : undefined }}>
                {l}
              </button>
            ))}
          </div>
          <div style={{ display: 'flex', gap: 8 }}>
            {TIME_RANGES.map(r => (
              <button key={r.value} onClick={() => setRange(r.value)}
                style={{ fontWeight: range === r.value ? 'bold' : 'normal' }}>{r.label}</button>
            ))}
          </div>
          <div style={{ display: 'flex', gap: 12, alignItems: 'center' }}>
            {kibanaLink && (
              <a href={kibanaLink} target="_blank" rel="noreferrer" style={{ fontSize: 13, color: '#0070d2' }}>
                Open in Kibana Discover →
              </a>
            )}
            <a href="/docs/elasticsearch" target="_blank" style={{ fontSize: 13, color: '#888' }}>
              Data sources
            </a>
          </div>
        </div>

        {deployedAt && (
          <p style={{ fontSize: 12, color: '#555', marginBottom: 12 }}>
            Red line = last deploy ({new Date(deployedAt).toLocaleString()})
          </p>
        )}

        {/* Charts */}
        <div style={{ marginBottom: 32 }}>
          <h3 style={{ marginBottom: 8 }}>Log events — {errors.total} total</h3>
          <ErrorsOverTime data={errors.buckets} deployedAt={deployedAt} />
        </div>
        <div style={{ marginBottom: 32 }}>
          <h3 style={{ marginBottom: 8 }}>Request volume</h3>
          <RequestVolume data={volume.buckets} deployedAt={deployedAt} />
        </div>

        {/* Errors by shopper */}
        {shoppers.length > 0 && (
          <div style={{ marginBottom: 32 }}>
            <h3 style={{ marginBottom: 8 }}>Errors by shopper (top {shoppers.length})</h3>
            <table style={{ borderCollapse: 'collapse', width: '100%', maxWidth: 600 }}>
              <thead>
                <tr style={{ borderBottom: '2px solid #eee', fontSize: 12, color: '#666', textAlign: 'left' }}>
                  <th style={{ padding: '4px 12px' }}>Shopper ID</th>
                  <th style={{ padding: '4px 12px', textAlign: 'right' }}>Errors</th>
                </tr>
              </thead>
              <tbody>
                {shoppers.map(({ shopperId, count }) => (
                  <tr key={shopperId} style={{ borderBottom: '1px solid #eee' }}>
                    <td style={{ padding: '6px 12px', fontFamily: 'monospace', fontSize: 13 }}>{shopperId || '(anonymous)'}</td>
                    <td style={{ padding: '6px 12px', textAlign: 'right', fontWeight: 'bold' }}>{count}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}

        {/* Raw log entries */}
        <div>
          <h3 style={{ marginBottom: 8 }}>Recent log entries ({rawLogs.length})</h3>
          {rawLogs.length === 0 ? (
            <p style={{ color: '#999' }}>No log entries for this filter.</p>
          ) : (
            <table style={{ borderCollapse: 'collapse', width: '100%', fontSize: 12 }}>
              <thead>
                <tr style={{ borderBottom: '2px solid #eee', color: '#666', textAlign: 'left' }}>
                  <th style={{ padding: '4px 8px', whiteSpace: 'nowrap' }}>Time</th>
                  <th style={{ padding: '4px 8px' }}>Level</th>
                  <th style={{ padding: '4px 8px' }}>Message</th>
                  <th style={{ padding: '4px 8px' }}>App</th>
                  <th style={{ padding: '4px 8px' }}>Shopper</th>
                </tr>
              </thead>
              <tbody>
                {rawLogs.map((entry, i) => (
                  <tr key={i} style={{ borderBottom: '1px solid #f5f5f5', verticalAlign: 'top' }}>
                    <td style={{ padding: '6px 8px', whiteSpace: 'nowrap', color: '#666' }}>
                      {new Date(entry.timestamp).toLocaleString()}
                    </td>
                    <td style={{ padding: '6px 8px' }}>
                      <span style={{
                        display: 'inline-block', padding: '1px 6px', borderRadius: 3, fontSize: 11, fontWeight: 600,
                        background: LEVEL_COLORS[entry.level] || '#888', color: '#fff'
                      }}>{entry.level}</span>
                    </td>
                    <td style={{ padding: '6px 8px', maxWidth: 500 }}>
                      <div style={{ fontFamily: 'monospace', wordBreak: 'break-word' }}>{entry.message}</div>
                      {entry.stackTrace && (
                        <details style={{ marginTop: 4 }}>
                          <summary style={{ cursor: 'pointer', color: '#c53030', fontSize: 11 }}>Stack trace</summary>
                          <pre style={{ margin: '4px 0', fontSize: 10, overflow: 'auto', background: '#fff5f5', padding: 6 }}>
                            {entry.stackTrace}
                          </pre>
                        </details>
                      )}
                      {entry.pageUrl && (
                        <div style={{ marginTop: 2, fontSize: 11, color: '#666', fontFamily: 'monospace', wordBreak: 'break-all' }}>
                          {entry.pageUrl}
                        </div>
                      )}
                    </td>
                    <td style={{ padding: '6px 8px', fontFamily: 'monospace', whiteSpace: 'nowrap' }}>
                      {entry.app ?? '—'}
                    </td>
                    <td style={{ padding: '6px 8px', fontFamily: 'monospace', whiteSpace: 'nowrap' }}>
                      {entry.shopperId ?? '—'}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          )}
        </div>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 2: Verify in browser** — navigate to `/components/{name}/logs`:
  - Change level — charts, shopper table, and raw entries all update
  - Reload — level restores from localStorage
  - Deploy marker appears when component has `metaData.updatedAt`
  - "Open in Kibana Discover →" link opens pre-filtered to the component
  - Stack trace expander works on entries with `error.stack_trace`

- [ ] **Step 3: Commit**

```bash
git add pages/components/
git commit -m "feat: logs sub-page with raw entries table, shopper breakdown, Kibana Discover link"
```

---

## Task 18: Connect-gd-auth Wiring + _app.js Auth Gate

**Files:**
- Modify: `pages/_app.js`
- Modify: `gasket.js` (add express middleware)

- [ ] **Step 1: Add connect-gd-auth middleware to gasket.js**

`connect-gd-auth` is CJS-only, so we use `createRequire` at the top of `gasket.js` and register it via an async plugin hook. `ssoRedirect` is built per-request so SSO sends the user back to the page they were trying to reach, not always `/`.

```js
// Add to the TOP of gasket.js (alongside other imports):
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const gdAuth = require('connect-gd-auth');

// Add to the plugins array:
{
  name: 'cds-admin-auth',
  hooks: {
    async middleware(gasket) {
      const { auth = {} } = gasket.config;
      return [
        // Build ssoRedirect per-request so SSO bounces the user back to the
        // exact page they were on (e.g. /components/myComp/logs) rather than /.
        (req, res, next) => {
          req._gdAuthOptions = {
            appName: 'cds-admin',
            verify: gdAuth.risk.low,
            types: ['jomax'],
            auth: ['basic'],
            host: auth.host,
            ssoRedirect: `${auth.ssoBase}/?realm=jomax&app=cds-admin&path=${encodeURIComponent(req.originalUrl)}`
          };
          gdAuth(req._gdAuthOptions)(req, res, next);
        }
      ];
    }
  }
}
```

- [ ] **Step 2: Implement pages/_app.js auth gate**

```jsx
// pages/_app.js
// When /auth/me returns 401, reload — the server-side connect-gd-auth middleware
// will redirect the browser to SSO.
import { useEffect, useState } from 'react';

export default function App({ Component, pageProps }) {
  const [authed, setAuthed] = useState(null);
  useEffect(() => {
    fetch('/auth/me').then(res => {
      if (res.status === 401) window.location.reload();
      else setAuthed(true);
    });
  }, []);

  if (authed === null) return null;
  return <Component {...pageProps} />;
}
```

- [ ] **Step 3: Add `NEXT_PUBLIC_HOSTNAME` to `.env.local.example`**

```bash
NEXT_PUBLIC_HOSTNAME=cds-admin.int.test-gdcorp.tools
```

- [ ] **Step 4: Verify auth gate in browser** — navigate to `http://localhost:3000` without an SSO cookie, confirm redirect to SSO.

- [ ] **Step 5: Commit**

```bash
git add pages/_app.js gasket.js
git commit -m "feat: connect-gd-auth SSO gate"
```

---

## Verification Checklist

- [ ] `npm run local` — dev server starts, AWS SSO check passes
- [ ] `npm test` — all tests pass
- [ ] `/` — list loads with `lastUpdated` column; search filters case-insensitively; alpha-sorted; env dropdown in top nav changes which env's components are shown
- [ ] As regular user, list shows ONLY components you're allowlisted on; super-admin sees all
- [ ] `/components/new` — create a test component; cds-auth entry has `{ jomax: [''], cert: [''], awsiam: [''] }` (not creator's username)
- [ ] `/components/{name}` (main page) — sparklines load; notes textarea auto-saves; tab bar renders with all 6 tabs; env dropdown is in the top nav (not the tab bar)
- [ ] `/components/{name}/manifest` — JSON editor validates live; Save works; History/Rollback work; i18n hash saves; Slack test fires
- [ ] `/components/{name}/auth` — add/remove jomax users; last-user removal blocked; "How cds-auth works →" links to docs; cert/awsiam show read-only
- [ ] `/docs/cds-auth` — page renders explaining jomax/cert/awsiam
- [ ] `/docs/cds-auth` — page renders explaining jomax/cert/awsiam
- [ ] `/docs/elasticsearch` — page renders explaining logs cluster fields, RUM cluster fields, with example documents for each
- [ ] `/components/{name}/rum` — Web Vitals charts with deployment marker at `metaData.updatedAt`; RUM config editor saves
- [ ] `/components/{name}/logs` — level filter (error/warn/info/all) changes charts AND raw log entries; filter persists in localStorage after page reload; deployment marker visible; request volume chart renders; raw log table shows last 100 entries with timestamp, level badge, message, app, shopper ID; stack trace expander works on entries with `error.stack_trace`; errors-by-shopper table appears when shoppers have errors; "Open in Kibana Discover →" deep-link opens pre-filtered to the component; "Data sources" link opens `/docs/elasticsearch`
- [ ] `/components/{name}/sync` — Copy prod→test; allowlist checked on DESTINATION env
- [ ] `/analytics` — component dropdown, time ranges, full charts, Kibana dashboard deep-links
- [ ] Each sub-page URL is directly linkable (test by opening in a new tab)
- [ ] Try accessing another team's component URL directly — 403
- [ ] Deploy to Katana dev — IAM writes succeed without cert env vars

---

## Revision Notes

**v7 changes (Elasticsearch docs + raw logs + shopper breakdown):**

- **Elasticsearch Data Sources section** — Added to the plan between the Switchboard data model and Access Control Model. Documents both clusters (logs + RUM) with key fields, which queries use each, and example documents.
- **`pages/docs/elasticsearch.js`** — New static docs page explaining the two clusters; linked from the Logs sub-page "Data sources" link. Added to Task 12 and file structure.
- **Raw log table** — `queryRawLogs` added to `clients/elasticsearch.js`; `GET /api/analytics/logs` added to `routes/analytics.js`; Logs sub-page renders last 100 entries with timestamp, level badge, message, app, page URL, shopper ID, and collapsible stack trace.
- **Errors-by-shopper breakdown** — `queryErrorsByUser` (`terms` agg on `client.user.id`) added to ES client; `GET /api/analytics/errors-by-user` added to analytics routes; Logs sub-page renders top shoppers by error count when present.
- **Kibana Discover deep-link** — `kibanaDiscover: { logsUrl, logsDataViewId }` added to all environments in `gasket.js`. `GET /api/config` now exposes it alongside `kibanaDashboards`. `buildKibanaDiscoverUrl` helper builds Rison-encoded URL with pre-filtered KQL query.
- **Level filter scope expanded** — The log-level filter now applies to both chart data and raw log entries, so all sections (charts, shopper table, raw entries) stay in sync.

**v6 changes:**

- **Global env selector in Layout** — The env dropdown (dev/test/prod) lives in the `Layout` nav and applies to all pages: component list, all 6 component detail sub-pages, and analytics. The list page already used `<Layout env={env} onEnvChange={setEnv}>` — all component detail pages now do the same. `ComponentNav` only receives `name` and `env` (for building correct tab URLs); it no longer owns the selector. This makes the selected environment globally visible and consistent, and correctly scopes the component list to the chosen env.

**v5 changes:**

- **Per-env Kibana dashboard URLs** — Removed `NEXT_PUBLIC_KIBANA_*` env vars (baked at build time). Kibana base URLs now live in `gasket.js` `environments` block as `kibanaDashboards: { logs, rum }`. Exposed to the client via `GET /api/config` (new `routes/config.js`). Pages fetch `/api/config` on mount and use `kibanaDashboards.logs/rum` state for Kibana deep-links.
- **Dynamic SSO redirect path** — `ssoRedirect` no longer hardcodes `path=/`. Auth middleware now builds it per-request: `${auth.ssoBase}/?realm=jomax&app=cds-admin&path=${encodeURIComponent(req.originalUrl)}`. `gasket.js` environments now store `auth.ssoBase` (URL without path) instead of a full `ssoRedirect` string.
- **ES API keys: local file vs. Katana secrets** — `.env.local.example` updated with explicit comments: locally, set `ELASTICSEARCH_LOGS_API_KEY` / `ELASTICSEARCH_RUM_API_KEY` in `.env.local`; in dev/test/prod, Katana injects them as env vars. The `clients/elasticsearch.js` code reads `process.env.*` in both cases.

**v4 changes (tabbed multi-page architecture):**

- **Tabbed sub-pages** — Component detail is now 6 deep-linkable pages: main (sparklines + notes), manifest, auth, rum, logs, sync. `ComponentNav` tab bar shared across all. Previous single-page dashboard design dropped.
- **`lastUpdated` on list page** — `GET /api/components` now returns `[{ name, lastUpdated }]` instead of `[name]`. List page shows a "Last updated" column.
- **cds-auth default changed** — New components get `{ jomax: [''], cert: [''], awsiam: [''] }` (not `{ jomax: [accountName], ... }`). Creator must explicitly add themselves on the Auth tab.
- **Log level filter** — Logs sub-page has error/warn/info/all filter. Persisted per-component in localStorage as `cds-admin:logLevel:{name}`. API route accepts `?level=` param forwarded to `queryErrors`.
- **Deployment markers** — RUM and Logs sub-pages fetch `metaData.updatedAt` and render a `ReferenceLine` on all charts so engineers can correlate releases with performance changes.
- **cds-auth docs page** — `/docs/cds-auth` explains jomax/cert/awsiam auth types; linked from Auth sub-page.
- **Accessibility improvements** — `AccessList` component now shows cert and awsiam as read-only sections (previously only jomax was shown).
- **`queryErrors` level param** — `clients/elasticsearch.js` `queryErrors()` now accepts `level` as 4th arg; `'all'` omits the log.level term filter.

**v3 changes (single-page dashboard + access control):**

- **Notes feature** — `registry.{name}.notes` (plain text, no history). `routes/notes.js`, `components/Notes.js` with debounced auto-save.
- **Super-admin bypass** — `cds-admin.admins` Switchboard key. Not editable via the app.
- **Read access gating** — reads now require allowlist membership (or super-admin). `lib/access-control.js` exposes `isSuperAdmin`, `canAccess`, `assertCanAccess`, `listAccessibleComponents`.
- **Component list filtering** — `GET /api/components` filters to components the caller can access.

**v2 fixes:**
- Jest ESM: `jest.unstable_mockModule()` instead of `jest.mock()`
- Sync allowlist check moved to `toEnv`
- Last-user lockout prevention
- CSRF: `require-same-origin.js` middleware
- Central error handler: `middleware/error-handler.js`
- `JsonEditor` stale closure fix: separate `onChange` and `onValidate` props
- Request volume ES query: `message.keyword` regex

- [ ] **Step 1: Add test**

Add to `test/routes/components.test.js`:

```js
test('POST /api/components/:name/sync copies manifest and checks allowlist on toEnv', async () => {
  const { getCdsAuthClient, getCdsClient } = await import('../../clients/switchboard.js');
  // allowlist on toEnv must pass
  const authGet = jest.fn().mockResolvedValue({ jomax: ['jroig'] });
  getCdsAuthClient.mockReturnValue({ get: authGet });
  const srcManifest = { engine: 'esm', js: 'abc.es.js', dependencies: {} };
  const srcGet = jest.fn().mockResolvedValue(srcManifest);
  const dstPut = jest.fn().mockResolvedValue();
  getCdsClient
    .mockImplementationOnce(() => ({ get: srcGet }))   // source env
    .mockImplementationOnce(() => ({ put: dstPut }));  // dest env

  const res = await request(app)
    .post('/api/components/componentA/sync')
    .send({ fromEnv: 'prod', toEnv: 'test' });

  expect(res.status).toBe(200);
  // Allowlist was checked on the destination env (test), not source (prod)
  expect(getCdsAuthClient).toHaveBeenCalledWith('test');
  expect(srcGet).toHaveBeenCalledWith('registry.componentA');
  expect(dstPut).toHaveBeenCalledWith('registry.componentA', srcManifest);
});

test('POST /api/components/:name/sync returns 403 when user not allowlisted on toEnv', async () => {
  const { getCdsAuthClient } = await import('../../clients/switchboard.js');
  getCdsAuthClient.mockReturnValue({ get: jest.fn().mockResolvedValue({ jomax: ['alice'] }) });

  const res = await request(app)
    .post('/api/components/componentA/sync')
    .send({ fromEnv: 'dev', toEnv: 'prod' });

  expect(res.status).toBe(403);
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Add sync route**

```js
router.post('/api/components/:name/sync', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const { fromEnv, toEnv } = req.body;
    // Check allowlist on the DESTINATION env — that's where the destructive write happens.
    // A user with dev access must NOT be able to push to prod.
    await assertCanAccess(name, req.user.accountName, toEnv);
    const manifest = await getCdsClient(fromEnv).get(`registry.${name}`);
    await getCdsClient(toEnv).put(`registry.${name}`, manifest);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});
```

- [ ] **Step 4: Create components/SyncPanel.js**

```jsx
// components/SyncPanel.js
import { useState } from 'react';

const ENVS = ['dev', 'test', 'prod'];

export default function SyncPanel({ componentName }) {
  const [status, setStatus] = useState('');
  const [busy, setBusy] = useState(false);

  async function handleSync(fromEnv, toEnv) {
    if (!confirm(`Copy ${componentName} from ${fromEnv} → ${toEnv}?`)) return;
    setBusy(true);
    setStatus('');
    const res = await fetch(`/api/components/${componentName}/sync`, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ fromEnv, toEnv })
    });
    setBusy(false);
    setStatus(res.ok ? `Synced ${fromEnv} → ${toEnv}` : 'Sync failed');
  }

  return (
    <div>
      <h3>Environment Sync</h3>
      {status && <p>{status}</p>}
      <table style={{ borderCollapse: 'collapse' }}>
        <thead>
          <tr>
            <th style={{ padding: '8px 16px' }}>From</th>
            <th style={{ padding: '8px 16px' }}>To</th>
            <th></th>
          </tr>
        </thead>
        <tbody>
          {ENVS.flatMap(from =>
            ENVS.filter(to => to !== from).map(to => (
              <tr key={`${from}-${to}`}>
                <td style={{ padding: '8px 16px' }}>{from}</td>
                <td style={{ padding: '8px 16px' }}>{to}</td>
                <td>
                  <button onClick={() => handleSync(from, to)} disabled={busy} style={{ fontSize: 12 }}>
                    Copy {from} → {to}
                  </button>
                </td>
              </tr>
            ))
          )}
        </tbody>
      </table>
    </div>
  );
}
```

- [ ] **Step 5: Fill in the "Environment Sync" card in [name].js** — replace Sync placeholder with `<SyncPanel componentName={name} />`

- [ ] **Step 6: Commit**

```bash
git add routes/components.js components/SyncPanel.js pages/components/[name].js
git commit -m "feat: environment sync for manifests"
```

---

## Task 9: i18n Sub-key

**Files:**
- Create: `routes/i18n.js`
- Create: `test/routes/i18n.test.js`
- Fill in "i18n Version" card in `pages/components/[name].js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/i18n.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();
const mockGetHistory = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockGet, put: mockPut, getHistory: mockGetHistory })),
  getCdsAuthClient: jest.fn(() => ({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/i18n.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components/:name/i18n returns hash string', async () => {
  mockGet.mockResolvedValue('607a011');
  const res = await request(app).get('/api/components/componentA/i18n?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ value: '607a011' });
});

test('GET /api/components/:name/i18n returns 404 when not set', async () => {
  mockGet.mockResolvedValue(null);
  const res = await request(app).get('/api/components/componentA/i18n?env=dev');
  expect(res.status).toBe(404);
});

test('PUT /api/components/:name/i18n updates hash', async () => {
  mockPut.mockResolvedValue();
  const res = await request(app)
    .put('/api/components/componentA/i18n?env=dev')
    .send({ value: 'abc1234' });
  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('registry.componentA.i18nVersion', 'abc1234');
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/routes/i18n.test.js
```

- [ ] **Step 3: Create routes/i18n.js**

```js
// routes/i18n.js
import express from 'express';
import { getCdsClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/components/:name/i18n', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const value = await getCdsClient(env).get(`registry.${req.params.name}.i18nVersion`);
    if (value == null) return res.status(404).json({ error: 'i18n not configured' });
    res.json({ value });
  } catch (err) { next(err); }
});

router.put('/api/components/:name/i18n', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.i18nVersion`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.get('/api/components/:name/i18n/history', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const history = await getCdsClient(env).getHistory(`registry.${req.params.name}.i18nVersion`);
    res.json(history ?? []);
  } catch (err) { next(err); }
});

router.post('/api/components/:name/i18n/rollback', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.i18nVersion`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
npm test -- test/routes/i18n.test.js
```

- [ ] **Step 5: Fill in the "i18n Version" card in [name].js**

Add state + fetch for i18n data, then replace the i18n placeholder:

```jsx
// Add to state:
const [i18nValue, setI18nValue] = useState(null);
const [i18nHistory, setI18nHistory] = useState([]);
const [i18nInput, setI18nInput] = useState('');

// Add useEffect:
useEffect(() => {
  if (!name || tab !== 'i18n') return;
  fetch(`/api/components/${name}/i18n?env=${env}`)
    .then(r => r.status === 404 ? null : r.json())
    .then(data => { setI18nValue(data?.value ?? null); setI18nInput(data?.value ?? ''); });
  fetch(`/api/components/${name}/i18n/history?env=${env}`)
    .then(r => r.json()).then(setI18nHistory);
}, [name, env, tab]);

// Replace i18n placeholder:
{tab === 'i18n' && (
  <div>
    <h3>i18n Version Hash</h3>
    {i18nValue === null && <p style={{ color: '#999' }}>Not configured for this component.</p>}
    <input
      value={i18nInput}
      onChange={e => setI18nInput(e.target.value)}
      placeholder="e.g. 607a011"
      style={{ width: 200, marginRight: 8 }}
    />
    <button onClick={async () => {
      await fetch(`/api/components/${name}/i18n?env=${env}`, {
        method: 'PUT',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ value: i18nInput })
      });
      setI18nValue(i18nInput);
    }}>Save</button>
    <h4 style={{ marginTop: 24 }}>History</h4>
    <HistoryTimeline
      history={i18nHistory}
      onRollback={async value => {
        await fetch(`/api/components/${name}/i18n/rollback?env=${env}`, {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ value })
        });
        setI18nValue(value);
        setI18nInput(value);
      }}
    />
  </div>
)}
```

- [ ] **Step 6: Commit**

```bash
git add routes/i18n.js test/routes/i18n.test.js pages/components/[name].js
git commit -m "feat: i18n sub-key view/edit/history/rollback"
```

---

## Task 10: RUM Sub-key

**Files:**
- Create: `routes/rum.js`
- Create: `test/routes/rum.test.js`

Same shape as i18n but JSON value + Monaco editor.

- [ ] **Step 1: Write failing tests**

```js
// test/routes/rum.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockGet, put: mockPut, getHistory: jest.fn().mockResolvedValue([]) })),
  getCdsAuthClient: jest.fn(() => ({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/rum.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components/:name/rum returns RUM config', async () => {
  const rumConfig = { pollerInterval: 100, pollerTimeout: 60000, targetDomSelector: '[data-testid="header"]' };
  mockGet.mockResolvedValue(rumConfig);
  const res = await request(app).get('/api/components/componentA/rum?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual(rumConfig);
});

test('PUT /api/components/:name/rum updates RUM config', async () => {
  mockPut.mockResolvedValue();
  const rumConfig = { pollerInterval: 200, pollerTimeout: 30000, targetDomSelector: '[data-testid="cta"]' };
  const res = await request(app)
    .put('/api/components/componentA/rum?env=dev')
    .send(rumConfig);
  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('registry.componentA.rum', rumConfig);
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create routes/rum.js**

```js
// routes/rum.js
import express from 'express';
import { getCdsClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/components/:name/rum', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const value = await getCdsClient(env).get(`registry.${req.params.name}.rum`);
    if (value == null) return res.status(404).json({ error: 'RUM not configured' });
    res.json(value);
  } catch (err) { next(err); }
});

router.put('/api/components/:name/rum', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.rum`, req.body);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.get('/api/components/:name/rum/history', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const history = await getCdsClient(env).getHistory(`registry.${req.params.name}.rum`);
    res.json(history ?? []);
  } catch (err) { next(err); }
});

router.post('/api/components/:name/rum/rollback', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsClient(env).put(`registry.${name}.rum`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Fill in the "RUM Config" card in [name].js** — same pattern as i18n tab but use `<JsonEditor>` instead of `<input>` for the value.

- [ ] **Step 6: Commit**

```bash
git add routes/rum.js test/routes/rum.test.js pages/components/[name].js
git commit -m "feat: RUM sub-key view/edit/history/rollback"
```

---

## Task 11: Access Tab (cds-auth allowlist)

**Files:**
- Create: `routes/access.js`
- Create: `test/routes/access.test.js`
- Create: `components/AccessList.js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/access.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsAuthClient: jest.fn(() => ({ get: mockGet, put: mockPut }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/access.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/access/:name returns allowlist entry', async () => {
  mockGet.mockResolvedValue({ jomax: ['jroig'], cert: [], awsiam: [] });
  const res = await request(app).get('/api/access/componentA?env=dev');
  expect(res.status).toBe(200);
  expect(res.body.jomax).toEqual(['jroig']);
});

test('PUT /api/access/:name updates jomax array', async () => {
  mockGet.mockResolvedValue({ jomax: ['jroig'], cert: [], awsiam: [] });
  mockPut.mockResolvedValue();
  const res = await request(app)
    .put('/api/access/componentA?env=dev')
    .send({ jomax: ['jroig', 'alice'] });
  expect(res.status).toBe(200);
  expect(mockPut).toHaveBeenCalledWith('componentA', { jomax: ['jroig', 'alice'], cert: [], awsiam: [] });
});

test('PUT /api/access/:name rejects empty jomax array (lockout prevention)', async () => {
  mockGet.mockResolvedValue({ jomax: ['jroig'], cert: [], awsiam: [] });
  const res = await request(app)
    .put('/api/access/componentA?env=dev')
    .send({ jomax: [] });
  expect(res.status).toBe(400);
  expect(mockPut).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create routes/access.js**

```js
// routes/access.js
import express from 'express';
import { getCdsAuthClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/access/:name', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const entry = await getCdsAuthClient(env).get(req.params.name);
    if (!entry) return res.status(404).json({ error: 'Component access entry not found' });
    res.json(entry);
  } catch (err) { next(err); }
});

router.put('/api/access/:name', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    const newJomax = Array.isArray(req.body.jomax) ? req.body.jomax : [];
    // Never allow the allowlist to be emptied — that locks everyone out.
    if (newJomax.length === 0) {
      return res.status(400).json({ error: 'Allowlist cannot be empty (at least one jomax user required)' });
    }
    const current = await getCdsAuthClient(env).get(name) ?? { jomax: [], cert: [], awsiam: [] };
    const updated = { ...current, jomax: newJomax };
    await getCdsAuthClient(env).put(name, updated);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Create components/AccessList.js**

```jsx
// components/AccessList.js
import { useState, useEffect } from 'react';

export default function AccessList({ componentName, env }) {
  const [entry, setEntry] = useState(null);
  const [newUser, setNewUser] = useState('');
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  useEffect(() => {
    fetch(`/api/access/${componentName}?env=${env}`)
      .then(r => r.json())
      .then(setEntry);
  }, [componentName, env]);

  async function save(newJomax) {
    setSaving(true);
    setError('');
    const res = await fetch(`/api/access/${componentName}?env=${env}`, {
      method: 'PUT',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ jomax: newJomax })
    });
    setSaving(false);
    if (res.ok) {
      setEntry(e => ({ ...e, jomax: newJomax }));
    } else {
      const body = await res.json();
      setError(body.error || 'Failed to update');
    }
  }

  if (!entry) return <p>Loading…</p>;

  const canRemove = entry.jomax.length > 1;  // never remove the last user

  return (
    <div>
      <h3>Jomax Access</h3>
      {error && <p style={{ color: 'red' }}>{error}</p>}
      <ul>
        {entry.jomax.map(user => (
          <li key={user} style={{ marginBottom: 4 }}>
            {user}
            <button
              onClick={() => save(entry.jomax.filter(u => u !== user))}
              disabled={saving || !canRemove}
              title={!canRemove ? 'Cannot remove the last user' : ''}
              style={{ marginLeft: 8, fontSize: 11 }}
            >
              Remove
            </button>
          </li>
        ))}
      </ul>
      <div style={{ marginTop: 12 }}>
        <input
          value={newUser}
          onChange={e => setNewUser(e.target.value)}
          placeholder="jomax username"
          style={{ marginRight: 8 }}
        />
        <button
          onClick={() => { save([...entry.jomax, newUser]); setNewUser(''); }}
          disabled={saving || !newUser}
        >
          Add User
        </button>
      </div>
      {entry.cert?.length > 0 && (
        <div style={{ marginTop: 16 }}>
          <h4>Cert (read-only)</h4>
          <ul>{entry.cert.map(c => <li key={c}>{c}</li>)}</ul>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 5: Fill in the "Access" card in [name].js** — replace Access placeholder with `<AccessList componentName={name} env={env} />`

- [ ] **Step 6: Run tests — expect PASS + commit**

```bash
npm test -- test/routes/access.test.js
git add routes/access.js test/routes/access.test.js components/AccessList.js pages/components/[name].js
git commit -m "feat: cds-auth allowlist management"
```

---

## Task 12: Alerts (Slack Webhook)

**Files:**
- Create: `routes/alerts.js`
- Create: `clients/slack.js`
- Create: `test/routes/alerts.test.js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/alerts.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockGet = jest.fn();
const mockPut = jest.fn();

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsAdminClient: jest.fn(() => ({ get: mockGet, put: mockPut })),
  getCdsAuthClient: jest.fn(() => ({ get: jest.fn().mockResolvedValue({ jomax: ['jroig'] }) }))
}));
jest.unstable_mockModule('../../clients/slack.js', () => ({
  sendSlackMessage: jest.fn().mockResolvedValue()
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/alerts.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/alerts/:name returns slack config', async () => {
  mockGet.mockResolvedValue({ slackWebhookUrl: 'https://hooks.slack.com/test' });
  const res = await request(app).get('/api/alerts/componentA?env=dev');
  expect(res.status).toBe(200);
  expect(res.body.slackWebhookUrl).toBe('https://hooks.slack.com/test');
});

test('POST /api/alerts/:name/test sends a slack message', async () => {
  mockGet.mockResolvedValue({ slackWebhookUrl: 'https://hooks.slack.com/test' });
  const { sendSlackMessage } = await import('../../clients/slack.js');
  const res = await request(app).post('/api/alerts/componentA/test?env=dev');
  expect(res.status).toBe(200);
  expect(sendSlackMessage).toHaveBeenCalledWith('https://hooks.slack.com/test', expect.stringContaining('componentA'));
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create clients/slack.js**

```js
// clients/slack.js
export async function sendSlackMessage(webhookUrl, text) {
  const res = await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ text })
  });
  if (!res.ok) throw new Error(`Slack webhook failed: ${res.status}`);
}
```

- [ ] **Step 4: Create routes/alerts.js**

```js
// routes/alerts.js
import express from 'express';
import { getCdsAdminClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';
import { sendSlackMessage } from '../clients/slack.js';

const router = express.Router();

router.get('/api/alerts/:name', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const config = await getCdsAdminClient(env).get(`config.${req.params.name}`);
    res.json(config ?? { slackWebhookUrl: null });
  } catch (err) { next(err); }
});

router.put('/api/alerts/:name', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    await getCdsAdminClient(env).put(`config.${name}`, { slackWebhookUrl: req.body.slackWebhookUrl });
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.post('/api/alerts/:name/test', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const config = await getCdsAdminClient(env).get(`config.${req.params.name}`);
    if (!config?.slackWebhookUrl) return res.status(400).json({ error: 'No Slack webhook configured' });
    await sendSlackMessage(config.slackWebhookUrl, `Test message from CDS Admin for ${req.params.name}`);
    res.json({ ok: true });
  } catch (err) { next(err); }
});

export default router;
```

- [ ] **Step 5: Run tests — expect PASS**

- [ ] **Step 6: Fill in the "Alerts (Slack)" card in [name].js** with webhook URL input, save button, and test button.

- [ ] **Step 7: Commit**

```bash
git add routes/alerts.js clients/slack.js test/routes/alerts.test.js pages/components/[name].js
git commit -m "feat: slack webhook alerts config"
```

---

## Task 13: Notes (plain text, no history)

**Files:**
- Create: `routes/notes.js`
- Create: `test/routes/notes.test.js`
- Create: `components/Notes.js`
- Modify: `pages/components/[name].js` (fill in Notes card)

- [ ] **Step 1: Write failing tests**

```js
// test/routes/notes.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

const mockCdsGet = jest.fn();
const mockCdsPut = jest.fn();
const mockCdsAuthGet = jest.fn().mockResolvedValue({ jomax: ['jroig'] });
const mockCdsAdminGet = jest.fn().mockResolvedValue({ jomax: [] });

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockCdsGet, put: mockCdsPut })),
  getCdsAuthClient: jest.fn(() => ({ get: mockCdsAuthGet })),
  getCdsAdminClient: jest.fn(() => ({ get: mockCdsAdminGet }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/notes.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components/:name/notes returns markdown string', async () => {
  mockCdsGet.mockResolvedValue('# Deployment notes\n\nRolled back due to LCP regression.');
  const res = await request(app).get('/api/components/componentA/notes?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ value: '# Deployment notes\n\nRolled back due to LCP regression.' });
});

test('GET /api/components/:name/notes returns empty when unset', async () => {
  mockCdsGet.mockResolvedValue(null);
  const res = await request(app).get('/api/components/componentA/notes?env=dev');
  expect(res.status).toBe(200);
  expect(res.body).toEqual({ value: '' });
});

test('PUT /api/components/:name/notes writes the markdown', async () => {
  mockCdsPut.mockResolvedValue();
  const res = await request(app)
    .put('/api/components/componentA/notes?env=dev')
    .send({ value: '## New notes' });
  expect(res.status).toBe(200);
  expect(mockCdsPut).toHaveBeenCalledWith('registry.componentA.notes', '## New notes');
});

test('PUT /api/components/:name/notes rejects non-string body', async () => {
  const res = await request(app)
    .put('/api/components/componentA/notes?env=dev')
    .send({ value: { not: 'a string' } });
  expect(res.status).toBe(400);
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npm test -- test/routes/notes.test.js
```

- [ ] **Step 3: Create routes/notes.js**

```js
// routes/notes.js
// Notes are stored as a markdown string in registry.{name}.notes.
// No history/rollback — just get/put. Caller must have access to the component.
import express from 'express';
import { getCdsClient } from '../clients/switchboard.js';
import { requireJomax } from '../middleware/require-jomax.js';
import { assertCanAccess } from '../lib/access-control.js';

const router = express.Router();

router.get('/api/components/:name/notes', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    await assertCanAccess(req.params.name, req.user.accountName, env);
    const value = await getCdsClient(env).get(`registry.${req.params.name}.notes`);
    res.json({ value: value ?? '' });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

router.put('/api/components/:name/notes', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertCanAccess(name, req.user.accountName, env);
    if (typeof req.body.value !== 'string') {
      return res.status(400).json({ error: 'value must be a string' });
    }
    await getCdsClient(env).put(`registry.${name}.notes`, req.body.value);
    res.json({ ok: true });
  } catch (err) {
    if (err.status) return res.status(err.status).json({ error: err.message });
    next(err);
  }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create components/Notes.js**

```jsx
// components/Notes.js
// Plain-text notes with debounced auto-save (1.5s after typing stops).
import { useState, useEffect, useRef } from 'react';

export default function Notes({ componentName, env }) {
  const [value, setValue] = useState('');
  const [status, setStatus] = useState('');
  const [loaded, setLoaded] = useState(false);
  const saveTimer = useRef(null);

  useEffect(() => {
    fetch(`/api/components/${componentName}/notes?env=${env}`)
      .then(r => r.json())
      .then(data => { setValue(data.value ?? ''); setLoaded(true); });
  }, [componentName, env]);

  useEffect(() => {
    if (!loaded) return;
    clearTimeout(saveTimer.current);
    setStatus('editing…');
    saveTimer.current = setTimeout(async () => {
      setStatus('saving…');
      const res = await fetch(`/api/components/${componentName}/notes?env=${env}`, {
        method: 'PUT',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ value })
      });
      setStatus(res.ok ? 'saved' : 'save failed');
      setTimeout(() => setStatus(''), 1500);
    }, 1500);
    return () => clearTimeout(saveTimer.current);
  }, [value]);

  if (!loaded) return <p>Loading…</p>;

  return (
    <div>
      <div style={{ display: 'flex', justifyContent: 'flex-end', fontSize: 12, color: '#666', marginBottom: 4 }}>
        {status}
      </div>
      <textarea
        value={value}
        onChange={e => setValue(e.target.value)}
        placeholder="Add notes…"
        style={{ width: '100%', minHeight: 180, fontFamily: 'monospace', fontSize: 13, padding: 8, border: '1px solid #ccc', borderRadius: 4, resize: 'vertical' }}
      />
    </div>
  );
}
```

- [ ] **Step 6: Fill in the Notes card on the dashboard**

In `pages/components/[name].js`:

```jsx
import Notes from '../../components/Notes.js';
// ...
<Card title="Notes">
  {name && <Notes componentName={name} env={env} />}
</Card>
```

- [ ] **Step 7: Commit**

```bash
git add routes/notes.js test/routes/notes.test.js components/Notes.js pages/components/[name].js
git commit -m "feat: plain-text notes per component (auto-save)"
```

---

## Task 14: Elasticsearch Client

**Files:**
- Create: `clients/elasticsearch.js`
- Create: `test/clients/elasticsearch.test.js`

- [ ] **Step 1: Write failing tests**

```js
// test/clients/elasticsearch.test.js
import { jest } from '@jest/globals';

const mockLogsSearch = jest.fn().mockResolvedValue({ hits: { total: { value: 5 } }, aggregations: {} });
const mockRumSearch = jest.fn().mockResolvedValue({ hits: { total: { value: 0 } }, aggregations: {} });

jest.unstable_mockModule('@elastic/elasticsearch', () => ({
  Client: jest.fn().mockImplementation(({ node }) => ({
    search: node.includes('rum') ? mockRumSearch : mockLogsSearch
  }))
}));

process.env.ELASTICSEARCH_LOGS_URL = 'https://logs.example.com';
process.env.ELASTICSEARCH_LOGS_API_KEY = 'bG9nczp0ZXN0';
process.env.ELASTICSEARCH_RUM_URL = 'https://rum.example.com';
process.env.ELASTICSEARCH_RUM_API_KEY = 'cnVtOnRlc3Q=';

const { getLogsClient, getRumClient, queryErrors, queryRequestVolume, queryWebVitals } = await import('../../clients/elasticsearch.js');

test('getLogsClient returns singleton', () => {
  const a = getLogsClient();
  const b = getLogsClient();
  expect(a).toBe(b);
});

test('getRumClient returns singleton distinct from logs client', () => {
  const logs = getLogsClient();
  const rum = getRumClient();
  expect(logs).not.toBe(rum);
});

test('queryErrors calls search on logs cluster with correct index', async () => {
  await queryErrors('myComponent', 'now-1d', 'now');
  expect(mockLogsSearch).toHaveBeenCalledWith(expect.objectContaining({
    index: '.ds-cds-api-prod-logs-*',
    body: expect.objectContaining({
      query: expect.objectContaining({
        bool: expect.objectContaining({
          must: expect.arrayContaining([
            { term: { 'log.level': 'error' } },
            { term: { 'event.type': 'myComponent' } }
          ])
        })
      })
    })
  }));
});

test('queryWebVitals calls search on RUM cluster with correct index', async () => {
  await queryWebVitals('myComponent', 'now-1d', 'now');
  expect(mockRumSearch).toHaveBeenCalledWith(expect.objectContaining({
    index: 'sns-traffic-c1-prod*'
  }));
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create clients/elasticsearch.js**

```js
// clients/elasticsearch.js
// Two separate Elastic clusters: logs (errors + volume) and RUM (web vitals).
// Each has its own URL and API key — see .env.local.example.
import { Client } from '@elastic/elasticsearch';

let logsClient;
let rumClient;

export function getLogsClient() {
  if (!logsClient) {
    logsClient = new Client({
      node: process.env.ELASTICSEARCH_LOGS_URL,
      auth: { apiKey: process.env.ELASTICSEARCH_LOGS_API_KEY }
    });
  }
  return logsClient;
}

export function getRumClient() {
  if (!rumClient) {
    rumClient = new Client({
      node: process.env.ELASTICSEARCH_RUM_URL,
      auth: { apiKey: process.env.ELASTICSEARCH_RUM_API_KEY }
    });
  }
  return rumClient;
}

export async function queryErrors(componentName, from, to) {
  return getLogsClient().search({
    index: '.ds-cds-api-prod-logs-*',
    body: {
      size: 0,
      query: {
        bool: {
          must: [
            { term: { 'log.level': 'error' } },
            { term: { 'event.type': componentName } },
            { range: { '@timestamp': { gte: from, lte: to } } }
          ]
        }
      },
      aggs: {
        over_time: {
          date_histogram: { field: '@timestamp', fixed_interval: '30m', min_doc_count: 0 }
        },
        total: { value_count: { field: '@timestamp' } }
      }
    }
  });
}

export async function queryRequestVolume(componentName, from, to) {
  // `componentTypes` can be a comma-separated list (e.g. componentTypes=foo,bar,baz).
  // We match the component appearing in ANY position via regex on the keyword subfield.
  // Examples that must match:
  //   ...componentTypes=bamMediaManagerEsm
  //   ...componentTypes=foo,bamMediaManagerEsm
  //   ...componentTypes=foo,bamMediaManagerEsm&other=x
  //   ...componentTypes=bamMediaManagerEsm,baz
  const escaped = componentName.replace(/[.?+*|{}[\]()"\\]/g, '\\$&');
  return getLogsClient().search({
    index: '.ds-cds-api-prod-logs-*',
    body: {
      size: 0,
      query: {
        bool: {
          must: [
            { term: { 'log.level': 'info' } },
            { regexp: { 'message.keyword': `.*componentTypes=([^&]*,)?${escaped}(,[^& ]*)?.*` } },
            { range: { '@timestamp': { gte: from, lte: to } } }
          ]
        }
      },
      aggs: {
        over_time: {
          date_histogram: { field: '@timestamp', fixed_interval: '1h', min_doc_count: 0 }
        }
      }
    }
  });
}

export async function queryWebVitals(componentName, from, to) {
  return getRumClient().search({
    index: 'sns-traffic-c1-prod*',
    body: {
      size: 0,
      query: {
        bool: {
          must: [
            { term: { 'customProps.media_library.componentType': componentName } },
            { range: { '@timestamp': { gte: from, lte: to } } }
          ]
        }
      },
      aggs: {
        over_time: {
          date_histogram: { field: '@timestamp', fixed_interval: '1h', min_doc_count: 0 },
          aggs: {
            lcp_p75: { percentiles: { field: 'lcp', percents: [75] } },
            cls_p75: { percentiles: { field: 'cls', percents: [75] } },
            inp_p75: { percentiles: { field: 'inp', percents: [75] } }
          }
        }
      }
    }
  });
}
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Commit**

```bash
git add clients/elasticsearch.js test/clients/elasticsearch.test.js
git commit -m "feat: elasticsearch client with error/volume/RUM queries"
```

---

## Task 15: Analytics API Routes + Page + Dashboard Mini-Charts

**Files:**
- Create: `routes/analytics.js`
- Create: `test/routes/analytics.test.js`
- Create: `components/charts/ErrorsOverTime.js`
- Create: `components/charts/RequestVolume.js`
- Create: `components/charts/WebVitals.js`
- Create: `pages/analytics.js`

- [ ] **Step 1: Write failing tests**

```js
// test/routes/analytics.test.js
import { jest } from '@jest/globals';
import request from 'supertest';
import express from 'express';

jest.unstable_mockModule('../../clients/elasticsearch.js', () => ({
  queryErrors: jest.fn().mockResolvedValue({
    hits: { total: { value: 12 } },
    aggregations: { over_time: { buckets: [] } }
  }),
  queryRequestVolume: jest.fn().mockResolvedValue({
    aggregations: { over_time: { buckets: [] } }
  }),
  queryWebVitals: jest.fn().mockResolvedValue({
    aggregations: { over_time: { buckets: [] } }
  })
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/analytics.js');
const app = express();
app.use(router);

test('GET /api/analytics/errors returns chart data and total', async () => {
  const res = await request(app).get('/api/analytics/errors?component=myComp&from=now-1d&to=now');
  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('total');
  expect(res.body).toHaveProperty('buckets');
});

test('GET /api/analytics/volume returns chart data', async () => {
  const res = await request(app).get('/api/analytics/volume?component=myComp&from=now-1d&to=now');
  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('buckets');
});

test('GET /api/analytics/vitals returns chart data', async () => {
  const res = await request(app).get('/api/analytics/vitals?component=myComp&from=now-1d&to=now');
  expect(res.status).toBe(200);
  expect(res.body).toHaveProperty('buckets');
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create routes/analytics.js**

```js
// routes/analytics.js
import express from 'express';
import { queryErrors, queryRequestVolume, queryWebVitals } from '../clients/elasticsearch.js';
import { requireJomax } from '../middleware/require-jomax.js';

const router = express.Router();

router.get('/api/analytics/errors', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now' } = req.query;
    const result = await queryErrors(component, from, to);
    res.json({
      total: result.hits.total.value,
      buckets: result.aggregations.over_time.buckets.map(b => ({
        time: b.key_as_string,
        count: b.doc_count
      }))
    });
  } catch (err) { next(err); }
});

router.get('/api/analytics/volume', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now' } = req.query;
    const result = await queryRequestVolume(component, from, to);
    res.json({
      buckets: result.aggregations.over_time.buckets.map(b => ({
        time: b.key_as_string,
        count: b.doc_count
      }))
    });
  } catch (err) { next(err); }
});

router.get('/api/analytics/vitals', requireJomax, async (req, res, next) => {
  try {
    const { component, from = 'now-1d', to = 'now' } = req.query;
    const result = await queryWebVitals(component, from, to);
    res.json({
      buckets: result.aggregations.over_time.buckets.map(b => ({
        time: b.key_as_string,
        lcp_p75: b.lcp_p75?.values?.['75.0'] ?? null,
        cls_p75: b.cls_p75?.values?.['75.0'] ?? null,
        inp_p75: b.inp_p75?.values?.['75.0'] ?? null
      }))
    });
  } catch (err) { next(err); }
});

export default router;
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Create chart components**

```jsx
// components/charts/ErrorsOverTime.js
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

export default function ErrorsOverTime({ data }) {
  return (
    <ResponsiveContainer width="100%" height={200}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tick={{ fontSize: 11 }} />
        <YAxis allowDecimals={false} />
        <Tooltip />
        <Line type="monotone" dataKey="count" stroke="#e53e3e" dot={false} />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```jsx
// components/charts/RequestVolume.js
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

export default function RequestVolume({ data }) {
  return (
    <ResponsiveContainer width="100%" height={200}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tick={{ fontSize: 11 }} />
        <YAxis allowDecimals={false} />
        <Tooltip />
        <Line type="monotone" dataKey="count" stroke="#38a169" dot={false} />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

```jsx
// components/charts/WebVitals.js
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

export default function WebVitals({ data }) {
  return (
    <ResponsiveContainer width="100%" height={250}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="time" tick={{ fontSize: 11 }} />
        <YAxis />
        <Tooltip />
        <Legend />
        <Line type="monotone" dataKey="lcp_p75" name="LCP p75 (ms)" stroke="#3182ce" dot={false} />
        <Line type="monotone" dataKey="inp_p75" name="INP p75 (ms)" stroke="#805ad5" dot={false} />
        <Line type="monotone" dataKey="cls_p75" name="CLS p75" stroke="#d69e2e" dot={false} />
      </LineChart>
    </ResponsiveContainer>
  );
}
```

- [ ] **Step 6: Create pages/analytics.js**

```jsx
// pages/analytics.js
import { useState, useEffect } from 'react';
import Layout from '../components/Layout.js';
import ErrorsOverTime from '../components/charts/ErrorsOverTime.js';
import RequestVolume from '../components/charts/RequestVolume.js';
import WebVitals from '../components/charts/WebVitals.js';

const TIME_RANGES = [
  { label: '1h',  value: 'now-1h' },
  { label: '6h',  value: 'now-6h' },
  { label: '24h', value: 'now-1d' },
  { label: '7d',  value: 'now-7d' }
];

// Build a Kibana deep-link that pre-filters the dashboard to a specific component.
// Kibana's _a param accepts a KQL query via URL state; encoding it lets the dashboard
// open already filtered without the user having to type anything.
function kibanaLink(baseUrl, kqlQuery) {
  if (!baseUrl) return null;
  const state = `(query:(language:kuery,query:'${kqlQuery}'))`;
  return `${baseUrl}&_a=${encodeURIComponent(state)}`;
}

export default function Analytics() {
  const [components, setComponents] = useState([]);
  const [component, setComponent] = useState('');
  const [range, setRange] = useState('now-1d');
  const [errors, setErrors] = useState({ total: 0, buckets: [] });
  const [volume, setVolume] = useState({ buckets: [] });
  const [vitals, setVitals] = useState({ buckets: [] });
  const [loading, setLoading] = useState(false);
  const [kibanaDashboards, setKibanaDashboards] = useState({});

  useEffect(() => {
    fetch('/api/config').then(r => r.json()).then(d => setKibanaDashboards(d.kibanaDashboards ?? {}));
    fetch('/api/components?env=prod').then(r => r.json()).then(names => {
      setComponents(names);
      if (names.length) setComponent(names[0]);
    });
  }, []);

  useEffect(() => {
    if (!component) return;
    setLoading(true);
    const params = `component=${component}&from=${range}&to=now`;
    Promise.all([
      fetch(`/api/analytics/errors?${params}`).then(r => r.json()),
      fetch(`/api/analytics/volume?${params}`).then(r => r.json()),
      fetch(`/api/analytics/vitals?${params}`).then(r => r.json())
    ]).then(([e, v, w]) => {
      setErrors(e);
      setVolume(v);
      setVitals(w);
      setLoading(false);
    });
  }, [component, range]);

  const logsLink = component ? kibanaLink(kibanaDashboards.logs, `event.type:"${component}"`) : null;
  const rumLink  = component ? kibanaLink(kibanaDashboards.rum,  `customProps.media_library.componentType:"${component}"`) : null;

  return (
    <Layout>
      <div style={{ padding: 24 }}>
        <h1>Analytics</h1>
        <div style={{ display: 'flex', gap: 16, marginBottom: 24, alignItems: 'center' }}>
          <select value={component} onChange={e => setComponent(e.target.value)}>
            {components.map(c => <option key={c} value={c}>{c}</option>)}
          </select>
          <div style={{ display: 'flex', gap: 8 }}>
            {TIME_RANGES.map(r => (
              <button
                key={r.value}
                onClick={() => setRange(r.value)}
                style={{ fontWeight: range === r.value ? 'bold' : 'normal' }}
              >
                {r.label}
              </button>
            ))}
          </div>
          {loading && <span style={{ color: '#999' }}>Loading…</span>}
        </div>

        <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: 24 }}>
          <div>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline' }}>
              <h3>Errors over time <span style={{ fontSize: 20, fontWeight: 'bold', marginLeft: 8 }}>{errors.total}</span> total</h3>
              {logsLink && <a href={logsLink} target="_blank" rel="noreferrer" style={{ fontSize: 12, color: '#0070d2' }}>Open in Kibana →</a>}
            </div>
            <ErrorsOverTime data={errors.buckets} />
          </div>
          <div>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline' }}>
              <h3>Request volume</h3>
              {logsLink && <a href={logsLink} target="_blank" rel="noreferrer" style={{ fontSize: 12, color: '#0070d2' }}>Open in Kibana →</a>}
            </div>
            <RequestVolume data={volume.buckets} />
          </div>
          <div style={{ gridColumn: '1 / -1' }}>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'baseline' }}>
              <h3>Web Vitals (p75)</h3>
              {rumLink && <a href={rumLink} target="_blank" rel="noreferrer" style={{ fontSize: 12, color: '#0070d2' }}>Open in Kibana →</a>}
            </div>
            <WebVitals data={vitals.buckets} />
          </div>
        </div>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 7: Create components/charts/MiniChart.js**

Compact sparkline shown in the dashboard's Analytics card — one chart per metric (errors, volume, LCP p75).

```jsx
// components/charts/MiniChart.js
import { LineChart, Line, YAxis, ResponsiveContainer, Tooltip } from 'recharts';

export default function MiniChart({ data, dataKey, color, label, unit = '' }) {
  const latest = data?.length ? data[data.length - 1][dataKey] : null;
  return (
    <div style={{ border: '1px solid #eee', borderRadius: 4, padding: 12 }}>
      <div style={{ fontSize: 11, color: '#666', textTransform: 'uppercase', letterSpacing: 0.5 }}>{label}</div>
      <div style={{ fontSize: 22, fontWeight: 'bold', marginBottom: 4 }}>
        {latest != null ? `${Math.round(latest)}${unit}` : '—'}
      </div>
      <ResponsiveContainer width="100%" height={60}>
        <LineChart data={data}>
          <YAxis hide domain={['auto', 'auto']} />
          <Tooltip />
          <Line type="monotone" dataKey={dataKey} stroke={color} strokeWidth={2} dot={false} />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

- [ ] **Step 8: Fill in the "Analytics — last 24h" card in pages/components/[name].js**

```jsx
import MiniChart from '../../components/charts/MiniChart.js';

// Inside the dashboard component — add state + effect:
const [miniErrors, setMiniErrors] = useState({ total: 0, buckets: [] });
const [miniVolume, setMiniVolume] = useState({ buckets: [] });
const [miniVitals, setMiniVitals] = useState({ buckets: [] });

useEffect(() => {
  if (!name) return;
  const p = `component=${name}&from=now-1d&to=now`;
  Promise.all([
    fetch(`/api/analytics/errors?${p}`).then(r => r.json()),
    fetch(`/api/analytics/volume?${p}`).then(r => r.json()),
    fetch(`/api/analytics/vitals?${p}`).then(r => r.json())
  ]).then(([e, v, w]) => {
    setMiniErrors(e);
    setMiniVolume(v);
    setMiniVitals(w);
  });
}, [name]);

// Replace the Analytics card:
<Card title="Analytics — last 24h" wide actions={
  <a href={`/analytics?component=${name}`} style={{ fontSize: 12, color: '#0070d2' }}>Full analytics →</a>
}>
  <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: 12 }}>
    <MiniChart data={miniErrors.buckets} dataKey="count" color="#e53e3e" label={`Errors (${miniErrors.total} total)`} />
    <MiniChart data={miniVolume.buckets} dataKey="count" color="#38a169" label="Request volume" />
    <MiniChart data={miniVitals.buckets} dataKey="lcp_p75" color="#3182ce" label="LCP p75" unit="ms" />
  </div>
</Card>
```

- [ ] **Step 8: Commit**

```bash
git add routes/analytics.js test/routes/analytics.test.js \
  components/charts/ pages/analytics.js
git commit -m "feat: analytics dashboard with errors, volume, and web vitals"
```

---

## Task 16: Connect-gd-auth Wiring + _app.js Auth Gate

**Files:**
- Modify: `pages/_app.js`
- Modify: `gasket.js` (add express middleware)

- [ ] **Step 1: Add connect-gd-auth middleware to gasket.js**

`connect-gd-auth` is CJS-only, so we use `createRequire` at the top of `gasket.js` and register the middleware via a plugin hook. The hook must be `async` because Gasket 7's `middleware` lifecycle is async. The `gasket` argument carries the per-environment `config.auth` object, so the SSO host is picked up automatically from the environments block — no env var needed.

```js
// Add to the TOP of gasket.js (alongside other imports):
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const gdAuth = require('connect-gd-auth');

// Add to the plugins array:
{
  name: 'cds-admin-auth',
  hooks: {
    async middleware(gasket) {
      const { auth = {} } = gasket.config;
      return [
        (req, res, next) => {
          gdAuth({
            appName: 'cds-admin',
            verify: gdAuth.risk.low,
            types: ['jomax'],
            auth: ['basic'],
            host: auth.host,
            ssoRedirect: `${auth.ssoBase}/?realm=jomax&app=cds-admin&path=${encodeURIComponent(req.originalUrl)}`
          })(req, res, next);
        }
      ];
    }
  }
}
```

- [ ] **Step 2: Implement pages/_app.js auth gate**

```jsx
// pages/_app.js
// When /auth/me returns 401, we reload — the server-side connect-gd-auth middleware
// redirects the browser to SSO with path= set to req.originalUrl so the user lands
// back on the exact page they came from after authenticating.
import { useEffect, useState } from 'react';

export default function App({ Component, pageProps }) {
  const [authed, setAuthed] = useState(null);
  useEffect(() => {
    fetch('/auth/me').then(res => {
      if (res.status === 401) {
        // Let the server-side middleware redirect to SSO by reloading
        window.location.reload();
      } else {
        setAuthed(true);
      }
    });
  }, []);

  if (authed === null) return null;  // avoid flash of unauthenticated content
  return <Component {...pageProps} />;
}
```

- [ ] **Step 2a: Also add `NEXT_PUBLIC_HOSTNAME` to `.env.local.example`**

```bash
NEXT_PUBLIC_HOSTNAME=cds-admin.int.test-gdcorp.tools
```

- [ ] **Step 3: Verify auth gate in browser**

```bash
npm run local
```

Navigate to `http://localhost:3000` — should redirect to SSO or (with valid cert/JWT) show the component list.

- [ ] **Step 4: Commit**

```bash
git add pages/_app.js gasket.js
git commit -m "feat: connect-gd-auth SSO gate"
```

---

## Verification Checklist

- [ ] `npm run local` — dev server starts, AWS SSO check passes
- [ ] `npm test` — all tests pass
- [ ] `/` — list loads with `lastUpdated` column; search filters case-insensitively; alpha-sorted; env dropdown in top nav changes which env's components are shown
- [ ] As regular user, list shows ONLY components you're allowlisted on; super-admin sees all
- [ ] `/components/new` — create a test component; cds-auth entry created; redirect to detail page
- [ ] `/components/{name}` (main page) — RUM sparklines + error sparklines + notes textarea load; notes auto-save after 1.5s; tab bar renders all 6 tabs
- [ ] `/components/{name}/manifest` — JSON editor validates live; Save works; History/Rollback work; i18n hash saves; Slack test fires
- [ ] `/components/{name}/auth` — add/remove jomax users; last-user removal blocked; "How cds-auth works →" links to docs; cert/awsiam show read-only
- [ ] `/docs/cds-auth` — page renders explaining jomax/cert/awsiam
- [ ] `/docs/elasticsearch` — page renders explaining logs cluster fields, RUM cluster fields, with example documents for each
- [ ] `/components/{name}/rum` — Web Vitals charts with deployment marker at `metaData.updatedAt`; RUM config editor saves
- [ ] `/components/{name}/logs` — level filter (error/warn/info/all) changes charts AND raw log entries; filter persists in localStorage after page reload; deployment marker visible; request volume chart renders; raw log table shows last 100 entries with timestamp, level badge, message, app, shopper ID; stack trace expander works on entries with `error.stack_trace`; errors-by-shopper table appears when there are shoppers; "Open in Kibana Discover →" deep-link opens pre-filtered to the component
- [ ] `/components/{name}/sync` — Copy prod→test; allowlist checked on DESTINATION env
- [ ] `/analytics` — component dropdown, time ranges, full charts, Kibana dashboard deep-links
- [ ] Each sub-page URL is directly linkable (test by opening in a new tab)
- [ ] Try accessing another team's component URL directly — 403
- [ ] Deploy to Katana dev — IAM writes succeed without cert env vars

---

## Revision Notes

**v7 changes (Elasticsearch docs + raw logs + shopper breakdown):**

- **Elasticsearch Data Sources section** — Added to the plan between the Switchboard data model and Access Control Model. Documents both clusters (logs + RUM) with key fields, which queries use each, and example documents. Engineers implementing the ES client now have a concrete reference for field names without having to grep live indices.
- **`pages/docs/elasticsearch.js`** — New static docs page explaining the two clusters; linked from the Logs sub-page. Added to file structure.
- **Raw log table** — `queryRawLogs` added to `clients/elasticsearch.js`; `GET /api/analytics/logs` added to `routes/analytics.js`; Logs sub-page renders last 100 entries with timestamp, level badge, message, app, page URL, shopper ID, and collapsible stack trace.
- **Errors-by-shopper breakdown** — `queryErrorsByUser` (`terms` agg on `client.user.id`) added to ES client; `GET /api/analytics/errors-by-user` added to analytics routes; Logs sub-page renders top shoppers by error count.
- **Kibana Discover deep-link** — `kibanaDiscover: { logsUrl, logsDataViewId }` added to all environments in `gasket.js`. `GET /api/config` now exposes it alongside `kibanaDashboards`. `buildKibanaDiscoverUrl` helper builds Rison-encoded URL with pre-filtered KQL query.
- **Level filter scope expanded** — The log-level filter now applies to both the chart data and the raw log entries, so all three sections (charts, shopper table, raw entries) stay in sync.
- **Verification checklist** — Updated from old single-page language to match current tabbed architecture; added specific checks for raw log table, shopper breakdown, Kibana Discover link, and docs pages.

**v6 changes (tabbed multi-page architecture):**

- **Global env selector in Layout** — The env dropdown (dev/test/prod) lives in the `Layout` nav and applies to all pages: component list, all 6 component detail sub-pages, and analytics. `ComponentNav` only receives `name` and `env` for building correct tab URLs; it no longer owns the selector.

**v5 changes:**

- **Per-env Kibana dashboard URLs** — Removed `NEXT_PUBLIC_KIBANA_*` env vars. Kibana base URLs now live in `gasket.js` `environments` block as `kibanaDashboards: { logs, rum }`. Exposed to the client via `GET /api/config`.
- **Dynamic SSO redirect path** — `ssoRedirect` now built per-request: `${auth.ssoBase}/?realm=jomax&app=cds-admin&path=${encodeURIComponent(req.originalUrl)}`.
- **ES API keys: local file vs. Katana secrets** — `.env.local.example` updated with explicit comments.

**v4 changes (tabbed sub-pages):**

- **Tabbed sub-pages** — Component detail is now 6 deep-linkable pages: main, manifest, auth, rum, logs, sync.
- **`lastUpdated` on list page** — `GET /api/components` returns `[{ name, lastUpdated }]`.
- **Log level filter** — Logs sub-page has error/warn/info/all filter; persisted in localStorage as `cds-admin:logLevel:{name}`.
- **Deployment markers** — RUM and Logs sub-pages fetch `metaData.updatedAt` and render a `ReferenceLine`.
- **cds-auth docs page** — `/docs/cds-auth` explains jomax/cert/awsiam auth types.
- **`queryErrors` level param** — accepts `level` as 4th arg; `'all'` omits the log.level term filter.

**v3 changes (single-page dashboard + access control):**

- **Single-page dashboard** — component detail page is now a grid of `Card` sections (Overview, Notes, Manifest, Access, Alerts, i18n, RUM, Sync, Recent History, Analytics) instead of tabs. Everything about a component is visible on one page.
- **Notes feature** — new `registry.{name}.notes` Switchboard key (plain text string, no history). New `routes/notes.js`, `components/Notes.js` with debounced auto-save. Task 13.
- **Super-admin bypass** — new `cds-admin.admins` Switchboard key (`{ jomax: [...] }`) lists users who bypass all per-component access checks. Not editable via the app.
- **Read access gating** — previously, any Jomax user could read any component. Now reads also require allowlist membership (or super-admin). New `lib/access-control.js` exposes `isSuperAdmin`, `canAccess`, `assertCanAccess`, `listAccessibleComponents`. All detail GETs call `assertCanAccess`.
- **Component list filtering** — `GET /api/components` filters to components the user is allowlisted on (super admins see all).
- **Prominent alpha-sorted search list** — `pages/index.js` gets a full-width search box, case-insensitive match anywhere in the name, locale-aware alpha sort.
- **Inline mini-charts** — Analytics card on the dashboard shows last-24h sparklines for errors, volume, LCP p75 with a link to `/analytics` for deeper dive.
- **File renames** — `lib/allowlist.js` → `lib/access-control.js`; `assertAllowlisted` → `assertCanAccess` everywhere.

**v2 fixes applied after review:**

- **Jest ESM mocking** — `jest.mock()` replaced with `jest.unstable_mockModule()` (only working API under `--experimental-vm-modules`). Added `jest.config.js` creation to Task 1.
- **Lazy Switchboard auth** — `buildClientOptions()` now called inside `getClient()` instead of at module load, so the module can be imported in SSR/test contexts without env vars set.
- **SWITCHBOARD_JWT branch added** — uses `fetchOptions.headers.Authorization: sso-jwt <token>` (may need adjustment based on actual client acceptance).
- **Sync allowlist check moved to `toEnv`** — prevents a user with only dev access from pushing to prod via sync.
- **Last-user lockout prevented** — access route rejects PUT that would empty the jomax array; UI disables Remove when only one user remains.
- **CSRF defense added** — `require-same-origin.js` middleware checks the `Origin`/`Referer` header on all state-changing requests.
- **Central error handler added** — `middleware/error-handler.js` prevents stack-trace leaks.
- **Connect-gd-auth middleware fixed** — `async middleware()` hook (was missing `async`); `createRequire` moved to top of `gasket.js`.
- **JsonEditor stale closure fixed** — `onChange` (value) and `onValidate` (valid flag) are now separate props; no shared state mutation.
- **AccessList useState → useEffect** — data fetch now uses proper effect hook with deps.
- **Request volume ES query fixed** — now uses `message.keyword` regex that matches component name in any position within the `componentTypes=foo,bar,baz` list.
- **Analytics decoupled from component detail** — removed as a tab; replaced with an `Analytics →` link to `/analytics?component={name}`.
- **Next.js imports** — dropped `.js` extensions from `next/router`, `next/dynamic`.
