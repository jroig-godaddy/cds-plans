# CDS Admin Tool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Gasket 7 / Next.js admin UI for GoDaddy's Component Delivery Service that lets engineers view and edit Switchboard manifests, manage access, roll back deployments, sync environments, and monitor component health via Elasticsearch.

**Architecture:** Gasket 7 app (ESM, Pages Router) with Express API routes backed by `@wsb/config-api-client` for Switchboard and `@elastic/elasticsearch` for analytics. All routes protected by `connect-gd-auth` Jomax SSO middleware. UI uses `@ux/*` GoDaddy component library + Monaco Editor for JSON editing + Recharts for analytics charts.

**Tech Stack:** Gasket 7, Next.js (Pages Router), Express, `@wsb/config-api-client`, `connect-gd-auth`, `@elastic/elasticsearch`, `@monaco-editor/react`, Recharts, `@ux/*`, Jest + Supertest

---

## File Structure

```
cds-admin/
├── gasket.js                        # Gasket 7 config: makeGasket + plugins
├── package.json                     # type:module, next/node scripts
├── next.config.js
├── .env.local.example
├── clients/
│   ├── switchboard.js               # ConfigClient factory — one instance per (app,env)
│   ├── elasticsearch.js             # ES client wrapper with query helpers
│   └── slack.js                     # Slack incoming webhook sender
├── plugins/
│   └── switchboard-auth.js          # Injects cert/IAM into ConfigClient at prepare
├── middleware/
│   └── require-jomax.js             # Reads req.gdAuth.jwt.jomax; 401 if missing
├── lib/
│   └── allowlist.js                 # Check/assert jomax user is in cds-auth allowlist
├── routes/
│   ├── auth.js                      # GET /auth/me, POST /auth/logout
│   ├── components.js                # registry.{name}: list, get, put, create, history, rollback, sync
│   ├── i18n.js                      # registry.{name}.i18nVersion: get, put, history, rollback
│   ├── rum.js                       # registry.{name}.rum: get, put, history, rollback
│   ├── access.js                    # cds-auth {name}: get, put jomax array
│   ├── alerts.js                    # cds-admin config.{name}: get, put, test
│   └── analytics.js                 # ES proxy: errors, volume, rum vitals
├── pages/
│   ├── _app.js                      # Auth gate — redirects to SSO if no jomax session
│   ├── index.js                     # Component list table
│   ├── components/
│   │   ├── new.js                   # Create component form
│   │   └── [name].js                # Tabbed detail: Manifest/History/Sync/i18n/RUM/Access/Alerts/Analytics
│   └── analytics.js                 # Standalone full analytics dashboard
├── components/
│   ├── Layout.js                    # Shell: nav, env selector, user display
│   ├── JsonEditor.js                # Monaco wrapper; exposes isValid + value
│   ├── HistoryTimeline.js           # List of history entries + Rollback button
│   ├── SyncPanel.js                 # Env status badges + Copy env→env buttons
│   ├── AccessList.js                # jomax list: add/remove usernames
│   └── charts/
│       ├── ErrorsOverTime.js        # Recharts LineChart for error count
│       ├── RequestVolume.js         # Recharts LineChart for request count
│       └── WebVitals.js             # Recharts LineChart for p75 LCP/CLS/INP
└── test/
    ├── clients/
    │   ├── switchboard.test.js
    │   └── elasticsearch.test.js
    ├── middleware/
    │   └── require-jomax.test.js
    ├── lib/
    │   └── allowlist.test.js
    └── routes/
        ├── components.test.js
        ├── i18n.test.js
        ├── rum.test.js
        ├── access.test.js
        ├── alerts.test.js
        └── analytics.test.js
```

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
  hostname: process.env.HOSTNAME || 'cds-admin.int.test-gdcorp.tools',
  environments: {
    local: { env: 'dev' },
    dev:   { env: 'dev' },
    test:  { env: 'test' },
    prod:  { env: 'prod' }
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
    "local": "GASKET_ENV=local next dev",
    "test":  "node --experimental-vm-modules node_modules/.bin/jest"
  }
}
```

- [ ] **Step 5: Create .env.local.example**

```bash
# .env.local.example
GASKET_ENV=local
ELASTICSEARCH_URL=https://usiessp-prod-usw2.kb.us-west-2.aws.found.io:9243
ELASTICSEARCH_API_KEY=

# Local dev auth — use ONE of:
SWITCHBOARD_CERT_PATH=/path/to/your.crt
SWITCHBOARD_CERT_KEY_PATH=/path/to/your.key
# OR
SWITCHBOARD_JWT=<token from ssojwt CLI>

# Production (injected by Katana)
# SWITCHBOARD_IAM_CERT_PATH=
# SWITCHBOARD_IAM_KEY_PATH=

SESSION_SECRET=dev-secret-change-in-prod
SSO_HOST=sso.int.test-gdcorp.tools
```

- [ ] **Step 6: Create jest.config.js** (required for ESM tests)

```js
// jest.config.js
export default {
  testEnvironment: 'node',
  transform: {},               // disable Babel — we're already ESM
  moduleFileExtensions: ['js', 'jsx', 'mjs'],
  testMatch: ['**/test/**/*.test.js']
};
```

- [ ] **Step 7: Verify dev server starts**

```bash
cp .env.local.example .env.local
# fill in your certs/jwt
npm run local
```

Expected: Next.js compiles, server starts on port 3000.

- [ ] **Step 8: Commit**

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
import { jest } from '@jest/globals';

// ESM: use jest.unstable_mockModule + dynamic imports AFTER the mock.
const ConfigClientMock = jest.fn().mockImplementation(() => ({
  get: jest.fn(),
  put: jest.fn(),
  getHistory: jest.fn()
}));

jest.unstable_mockModule('@wsb/config-api-client', () => ({ default: ConfigClientMock }));
jest.unstable_mockModule('@wsb/core-config-rules', () => ({ default: {} }));

process.env.SWITCHBOARD_CERT_PATH = '/fake/cert.crt';
process.env.SWITCHBOARD_CERT_KEY_PATH = '/fake/cert.key';

const { getCdsClient, getCdsAuthClient, getCdsAdminClient } = await import('../../clients/switchboard.js');

test('getCdsClient returns same instance for same env', () => {
  const a = getCdsClient('dev');
  const b = getCdsClient('dev');
  expect(a).toBe(b);
});

test('getCdsAuthClient uses cds-auth app', () => {
  getCdsAuthClient('dev');
  expect(ConfigClientMock).toHaveBeenCalledWith(expect.objectContaining({ app: 'cds-auth' }));
});

test('getCdsAdminClient uses cds-admin app', () => {
  getCdsAdminClient('dev');
  expect(ConfigClientMock).toHaveBeenCalledWith(expect.objectContaining({ app: 'cds-admin' }));
});

test('throws at first getClient call when no auth env vars set', async () => {
  delete process.env.SWITCHBOARD_CERT_PATH;
  delete process.env.SWITCHBOARD_CERT_KEY_PATH;
  delete process.env.SWITCHBOARD_IAM_CERT_PATH;
  delete process.env.SWITCHBOARD_JWT;
  jest.resetModules();
  jest.unstable_mockModule('@wsb/config-api-client', () => ({ default: ConfigClientMock }));
  jest.unstable_mockModule('@wsb/core-config-rules', () => ({ default: {} }));
  const mod = await import('../../clients/switchboard.js');
  expect(() => mod.getCdsClient('dev')).toThrow('No Switchboard auth');
});

test('uses JWT via fetchOptions header when SWITCHBOARD_JWT set', async () => {
  delete process.env.SWITCHBOARD_CERT_PATH;
  process.env.SWITCHBOARD_JWT = 'jwt-token-xyz';
  jest.resetModules();
  jest.unstable_mockModule('@wsb/config-api-client', () => ({ default: ConfigClientMock }));
  jest.unstable_mockModule('@wsb/core-config-rules', () => ({ default: {} }));
  const mod = await import('../../clients/switchboard.js');
  mod.getCdsClient('dev');
  expect(ConfigClientMock).toHaveBeenCalledWith(expect.objectContaining({
    fetchOptions: { headers: { Authorization: 'sso-jwt jwt-token-xyz' } }
  }));
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
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const ConfigClient = require('@wsb/config-api-client');
const configRules = require('@wsb/core-config-rules');

// Build auth config from env. Called lazily on first getClient() so the module
// can be imported in contexts (SSR, tests) without auth vars set.
function buildClientOptions() {
  // IAM cert (Katana prod)
  if (process.env.SWITCHBOARD_IAM_CERT_PATH) {
    return { auth: {
      cert: [process.env.SWITCHBOARD_IAM_CERT_PATH],
      key: process.env.SWITCHBOARD_IAM_KEY_PATH
    }};
  }
  // Client cert (local dev Option A)
  if (process.env.SWITCHBOARD_CERT_PATH) {
    return { auth: {
      cert: [process.env.SWITCHBOARD_CERT_PATH],
      key: process.env.SWITCHBOARD_CERT_KEY_PATH
    }};
  }
  // ssojwt bearer token (local dev Option B)
  if (process.env.SWITCHBOARD_JWT) {
    return { fetchOptions: {
      headers: { Authorization: `sso-jwt ${process.env.SWITCHBOARD_JWT}` }
    }};
  }
  throw new Error('No Switchboard auth configured. Set SWITCHBOARD_IAM_CERT_PATH, SWITCHBOARD_CERT_PATH, or SWITCHBOARD_JWT.');
}

const cache = {};

function getClient(app, env) {
  const key = `${app}:${env}`;
  if (!cache[key]) {
    cache[key] = new ConfigClient({
      app,
      env: env === 'local' ? 'dev' : env,
      plugins: [configRules],
      cache: { path: null, tts: 30000 },
      ...buildClientOptions()
    });
  }
  return cache[key];
}

export const getCdsClient      = env => getClient('cds', env);
export const getCdsAuthClient  = env => getClient('cds-auth', env);
export const getCdsAdminClient = env => getClient('cds-admin', env);
```

> **Note:** the `fetchOptions.headers.Authorization` approach for SWITCHBOARD_JWT assumes the underlying `@wsb/config-api-client` accepts a pre-built JWT via a Bearer-style header. If in practice it only accepts cert/key or IAM, this branch should be adjusted — the closest alternative is to use ssojwt's cert output as `SWITCHBOARD_CERT_PATH`.

- [ ] **Step 4: Run test — expect PASS**

```bash
npm test -- test/clients/switchboard.test.js
```

- [ ] **Step 5: Create plugins/switchboard-auth.js** (no test needed — pure config injection)

```js
// plugins/switchboard-auth.js
export default {
  name: 'cds-admin-switchboard-auth',
  hooks: {
    prepare(gasket) {
      // Validate auth env vars at startup so the app fails fast locally
      if (!process.env.SWITCHBOARD_IAM_CERT_PATH && !process.env.SWITCHBOARD_CERT_PATH) {
        gasket.logger.warn('No Switchboard cert configured. API routes will fail.');
      }
    }
  }
};
```

- [ ] **Step 6: Commit**

```bash
git add clients/switchboard.js plugins/switchboard-auth.js test/clients/switchboard.test.js
git commit -m "feat: switchboard client factory with cert/IAM auth"
```

---

## Task 3: Auth Middleware + Auth Route + Error Handler + CSRF

**Files:**
- Create: `middleware/require-jomax.js`
- Create: `middleware/require-same-origin.js` (CSRF defense)
- Create: `middleware/error-handler.js`
- Create: `lib/allowlist.js`
- Create: `routes/auth.js`
- Create: `test/middleware/require-jomax.test.js`
- Create: `test/middleware/require-same-origin.test.js`
- Create: `test/lib/allowlist.test.js`

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
// test/lib/allowlist.test.js
import { jest } from '@jest/globals';

jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsAuthClient: jest.fn(() => ({
    get: jest.fn().mockResolvedValue({ jomax: ['jroig', 'alice'] })
  }))
}));

const { assertAllowlisted } = await import('../../lib/allowlist.js');

test('resolves when user is in jomax allowlist', async () => {
  await expect(assertAllowlisted('componentA', 'jroig', 'dev')).resolves.toBeUndefined();
});

test('throws 403 when user is not in allowlist', async () => {
  await expect(assertAllowlisted('componentA', 'bob', 'dev')).rejects.toMatchObject({ status: 403 });
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

- [ ] **Step 4: Implement lib/allowlist.js**

```js
// lib/allowlist.js
import { getCdsAuthClient } from '../clients/switchboard.js';

export async function assertAllowlisted(componentName, accountName, env) {
  const client = getCdsAuthClient(env);
  const entry = await client.get(componentName);
  const allowed = entry?.jomax ?? [];
  if (!allowed.includes(accountName)) {
    const err = new Error(`${accountName} is not in the allowlist for ${componentName}`);
    err.status = 403;
    throw err;
  }
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

- [ ] **Step 8: Wire middleware in gasket.js**

Add a plugin hook that registers `requireSameOrigin` globally and `errorHandler` as the final handler:

```js
// Add to gasket.js plugins array:
{
  name: 'cds-admin-global-middleware',
  hooks: {
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

- [ ] **Step 9: Run tests — expect PASS**

```bash
npm test -- test/middleware test/lib
```

- [ ] **Step 10: Commit**

```bash
git add middleware/ lib/ routes/auth.js test/middleware/ test/lib/ gasket.js
git commit -m "feat: jomax auth, allowlist, csrf, error handler"
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

const mockGet = jest.fn();
jest.unstable_mockModule('../../clients/switchboard.js', () => ({
  getCdsClient: jest.fn(() => ({ get: mockGet, put: jest.fn() })),
  getCdsAuthClient: jest.fn(() => ({ put: jest.fn() })),
  getCdsAdminClient: jest.fn(() => ({ put: jest.fn() }))
}));
jest.unstable_mockModule('../../middleware/require-jomax.js', () => ({
  requireJomax: (req, res, next) => { req.user = { accountName: 'jroig' }; next(); }
}));

const { default: router } = await import('../../routes/components.js');
const app = express();
app.use(express.json());
app.use(router);

test('GET /api/components returns list of component names', async () => {
  mockGet.mockResolvedValue({
    'registry.componentA': {},
    'registry.componentB': {},
    'registry.componentA.rum': {},      // sub-keys excluded
    'registry.componentA.i18nVersion': {} // sub-keys excluded
  });

  const res = await request(app).get('/api/components');
  expect(res.status).toBe(200);
  expect(res.body).toEqual(['componentA', 'componentB']);
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
  expect(authClient.put).toHaveBeenCalledWith('newComponent', { jomax: ['jroig'], cert: [], awsiam: [] });
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
import { assertAllowlisted } from '../lib/allowlist.js';

const router = express.Router();

// Strip sub-keys (.rum, .i18nVersion) — only top-level registry.* keys
function extractComponentNames(allKeys) {
  return Object.keys(allKeys)
    .filter(k => k.startsWith('registry.') && k.split('.').length === 2)
    .map(k => k.replace('registry.', ''));
}

router.get('/api/components', requireJomax, async (req, res, next) => {
  try {
    const env = req.query.env || 'dev';
    const all = await getCdsClient(env).get();
    res.json(extractComponentNames(all ?? {}));
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
      getCdsAuthClient(env).put(name, { jomax: [req.user.accountName], cert: [], awsiam: [] }),
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
import { useState, useEffect } from 'react';
import Layout from '../components/Layout.js';

export default function ComponentList() {
  const [components, setComponents] = useState([]);
  const [env, setEnv] = useState('dev');
  const [filter, setFilter] = useState('');

  useEffect(() => {
    fetch(`/api/components?env=${env}`)
      .then(r => r.json())
      .then(setComponents);
  }, [env]);

  const filtered = components.filter(c => c.toLowerCase().includes(filter.toLowerCase()));

  return (
    <Layout env={env} onEnvChange={setEnv}>
      <div style={{ padding: '24px' }}>
        <h1>Components</h1>
        <input
          placeholder="Filter..."
          value={filter}
          onChange={e => setFilter(e.target.value)}
          style={{ marginBottom: 16, width: 300 }}
        />
        <a href="/components/new" style={{ float: 'right' }}>+ New Component</a>
        <table style={{ width: '100%', borderCollapse: 'collapse' }}>
          <thead>
            <tr>
              <th style={{ textAlign: 'left', padding: '8px', borderBottom: '1px solid #ccc' }}>Component</th>
            </tr>
          </thead>
          <tbody>
            {filtered.map(name => (
              <tr key={name}>
                <td style={{ padding: '8px', borderBottom: '1px solid #eee' }}>
                  <a href={`/components/${name}`}>{name}</a>
                </td>
              </tr>
            ))}
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
      router.push(`/components/${name}`);
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

## Task 6: Manifest Get/Edit + Monaco JSON Editor

**Files:**
- Add to `routes/components.js`: GET + PUT `:name`
- Create: `components/JsonEditor.js`
- Create: `pages/components/[name].js` (Manifest tab only for now)

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
    const manifest = await getCdsClient(env).get(`registry.${req.params.name}`);
    if (!manifest) return res.status(404).json({ error: 'Component not found' });
    res.json(manifest);
  } catch (err) { next(err); }
});

router.put('/api/components/:name', requireJomax, async (req, res, next) => {
  try {
    const { name } = req.params;
    const env = req.query.env || 'dev';
    await assertAllowlisted(name, req.user.accountName, env);
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

- [ ] **Step 6: Create pages/components/[name].js (Manifest tab)**

```jsx
// pages/components/[name].js
import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import Layout from '../../components/Layout.js';
import JsonEditor from '../../components/JsonEditor.js';

const TABS = ['Manifest', 'History', 'Sync', 'i18n', 'RUM', 'Access', 'Alerts'];

export default function ComponentDetail() {
  const router = useRouter();
  const { name } = router.query;
  const [env, setEnv] = useState('dev');
  const [tab, setTab] = useState('Manifest');
  const [manifest, setManifest] = useState(null);
  const [editorValue, setEditorValue] = useState('');
  const [editorValid, setEditorValid] = useState(true);
  const [saving, setSaving] = useState(false);
  const [saveMsg, setSaveMsg] = useState('');

  useEffect(() => {
    if (!name) return;
    fetch(`/api/components/${name}?env=${env}`)
      .then(r => r.json())
      .then(data => {
        setManifest(data);
        setEditorValue(JSON.stringify(data, null, 2));
      });
  }, [name, env]);

  async function handleSave() {
    setSaving(true);
    setSaveMsg('');
    const res = await fetch(`/api/components/${name}?env=${env}`, {
      method: 'PUT',
      headers: { 'content-type': 'application/json' },
      body: editorValue
    });
    setSaving(false);
    setSaveMsg(res.ok ? 'Saved' : 'Error saving');
  }

  return (
    <Layout env={env} onEnvChange={setEnv}>
      <div style={{ padding: 24 }}>
        <a href="/">← Back</a>
        <h1>{name}</h1>
        <div style={{ display: 'flex', gap: 8, marginBottom: 16, borderBottom: '1px solid #ccc' }}>
          {TABS.map(t => (
            <button
              key={t}
              onClick={() => setTab(t)}
              style={{
                padding: '8px 16px',
                border: 'none',
                borderBottom: tab === t ? '2px solid #0070d2' : '2px solid transparent',
                background: 'none',
                cursor: 'pointer',
                fontWeight: tab === t ? 'bold' : 'normal'
              }}
            >
              {t}
            </button>
          ))}
          <a
            href={`/analytics?component=${name}`}
            style={{ marginLeft: 'auto', padding: '8px 16px', color: '#0070d2' }}
          >
            Analytics →
          </a>
        </div>

        {tab === 'Manifest' && (
          <div>
            <JsonEditor
              value={editorValue}
              onChange={v => setEditorValue(v ?? '')}
              onValidate={setEditorValid}
            />
            <div style={{ marginTop: 8, display: 'flex', gap: 8, alignItems: 'center' }}>
              <button onClick={handleSave} disabled={!editorValid || saving}>
                {saving ? 'Saving…' : 'Save'}
              </button>
              {saveMsg && <span>{saveMsg}</span>}
            </div>
          </div>
        )}
        {tab !== 'Manifest' && <p style={{ color: '#999' }}>{tab} — wired up in later tasks</p>}
      </div>
    </Layout>
  );
}
```

- [ ] **Step 7: Verify in browser** — navigate to a component, edit JSON, confirm save button disables on invalid JSON.

- [ ] **Step 8: Commit**

```bash
git add routes/components.js components/JsonEditor.js pages/components/[name].js test/routes/components.test.js
git commit -m "feat: manifest view/edit with Monaco JSON validation"
```

---

## Task 7: History + Rollback

**Files:**
- Add to `routes/components.js`: GET `:name/history`, POST `:name/rollback`
- Create: `components/HistoryTimeline.js`
- Wire History tab in `pages/components/[name].js`

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
    await assertAllowlisted(name, req.user.accountName, env);
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

- [ ] **Step 6: Wire History tab in [name].js**

In `pages/components/[name].js`, add state and data fetching for history, then replace the History placeholder:

```jsx
// Add to state:
const [history, setHistory] = useState([]);
const [historyLoading, setHistoryLoading] = useState(false);

// Add useEffect (alongside manifest fetch):
useEffect(() => {
  if (!name || tab !== 'History') return;
  setHistoryLoading(true);
  fetch(`/api/components/${name}/history?env=${env}`)
    .then(r => r.json())
    .then(data => { setHistory(data); setHistoryLoading(false); });
}, [name, env, tab]);

// Replace the History placeholder with:
{tab === 'History' && (
  <HistoryTimeline
    history={history}
    loading={historyLoading}
    onRollback={async value => {
      await fetch(`/api/components/${name}/rollback?env=${env}`, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ value })
      });
      // Refresh manifest after rollback
      fetch(`/api/components/${name}?env=${env}`).then(r => r.json()).then(setManifest);
    }}
  />
)}
```

- [ ] **Step 7: Commit**

```bash
git add routes/components.js components/HistoryTimeline.js pages/components/[name].js
git commit -m "feat: history and rollback for manifests"
```

---

## Task 8: Environment Sync

**Files:**
- Add to `routes/components.js`: POST `:name/sync`
- Create: `components/SyncPanel.js`

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
    await assertAllowlisted(name, req.user.accountName, toEnv);
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

- [ ] **Step 5: Wire Sync tab in [name].js** — replace Sync placeholder with `<SyncPanel componentName={name} />`

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
- Wire i18n tab in `pages/components/[name].js`

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
import { assertAllowlisted } from '../lib/allowlist.js';

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
    await assertAllowlisted(name, req.user.accountName, env);
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
    await assertAllowlisted(name, req.user.accountName, env);
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

- [ ] **Step 5: Wire i18n tab in [name].js**

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
import { assertAllowlisted } from '../lib/allowlist.js';

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
    await assertAllowlisted(name, req.user.accountName, env);
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
    await assertAllowlisted(name, req.user.accountName, env);
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

- [ ] **Step 5: Wire RUM tab in [name].js** — same pattern as i18n tab but use `<JsonEditor>` instead of `<input>` for the value.

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
import { assertAllowlisted } from '../lib/allowlist.js';

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
    await assertAllowlisted(name, req.user.accountName, env);
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

- [ ] **Step 5: Wire Access tab in [name].js** — replace Access placeholder with `<AccessList componentName={name} env={env} />`

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
import { assertAllowlisted } from '../lib/allowlist.js';
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
    await assertAllowlisted(name, req.user.accountName, env);
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

- [ ] **Step 6: Wire Alerts tab in [name].js** with webhook URL input, save button, and test button.

- [ ] **Step 7: Commit**

```bash
git add routes/alerts.js clients/slack.js test/routes/alerts.test.js pages/components/[name].js
git commit -m "feat: slack webhook alerts config"
```

---

## Task 13: Elasticsearch Client

**Files:**
- Create: `clients/elasticsearch.js`
- Create: `test/clients/elasticsearch.test.js`

- [ ] **Step 1: Write failing tests**

```js
// test/clients/elasticsearch.test.js
import { jest } from '@jest/globals';

jest.unstable_mockModule('@elastic/elasticsearch', () => ({
  Client: jest.fn().mockImplementation(() => ({
    search: jest.fn().mockResolvedValue({ hits: { total: { value: 5 } }, aggregations: {} })
  }))
}));

process.env.ELASTICSEARCH_URL = 'https://es.example.com';
process.env.ELASTICSEARCH_API_KEY = 'dGVzdDp0ZXN0';

const { getEsClient, queryErrors, queryRequestVolume, queryWebVitals } = await import('../../clients/elasticsearch.js');

test('getEsClient returns singleton', () => {
  const a = getEsClient();
  const b = getEsClient();
  expect(a).toBe(b);
});

test('queryErrors calls search on correct index with event.type filter', async () => {
  const client = getEsClient();
  await queryErrors('myComponent', 'now-1d', 'now');
  expect(client.search).toHaveBeenCalledWith(expect.objectContaining({
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

test('queryWebVitals calls search on RUM index with componentType filter', async () => {
  const client = getEsClient();
  await queryWebVitals('myComponent', 'now-1d', 'now');
  expect(client.search).toHaveBeenCalledWith(expect.objectContaining({
    index: 'sns-traffic-c1-prod*'
  }));
});
```

- [ ] **Step 2: Run test — expect FAIL**

- [ ] **Step 3: Create clients/elasticsearch.js**

```js
// clients/elasticsearch.js
import { Client } from '@elastic/elasticsearch';

let esClient;

export function getEsClient() {
  if (!esClient) {
    esClient = new Client({
      node: process.env.ELASTICSEARCH_URL,
      auth: { apiKey: process.env.ELASTICSEARCH_API_KEY }
    });
  }
  return esClient;
}

export async function queryErrors(componentName, from, to) {
  return getEsClient().search({
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
  return getEsClient().search({
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
  return getEsClient().search({
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

## Task 14: Analytics API Routes + Page

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

export default function Analytics() {
  const [components, setComponents] = useState([]);
  const [component, setComponent] = useState('');
  const [range, setRange] = useState('now-1d');
  const [errors, setErrors] = useState({ total: 0, buckets: [] });
  const [volume, setVolume] = useState({ buckets: [] });
  const [vitals, setVitals] = useState({ buckets: [] });
  const [loading, setLoading] = useState(false);

  useEffect(() => {
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
            <h3>Errors over time <span style={{ fontSize: 20, fontWeight: 'bold', marginLeft: 8 }}>{errors.total}</span> total</h3>
            <ErrorsOverTime data={errors.buckets} />
          </div>
          <div>
            <h3>Request volume</h3>
            <RequestVolume data={volume.buckets} />
          </div>
          <div style={{ gridColumn: '1 / -1' }}>
            <h3>Web Vitals (p75)</h3>
            <WebVitals data={vitals.buckets} />
          </div>
        </div>
      </div>
    </Layout>
  );
}
```

- [ ] **Step 7: Wire Analytics tab in [name].js** — link tab to `/analytics?component={name}` (new page, not inline tab).

- [ ] **Step 8: Commit**

```bash
git add routes/analytics.js test/routes/analytics.test.js \
  components/charts/ pages/analytics.js
git commit -m "feat: analytics dashboard with errors, volume, and web vitals"
```

---

## Task 15: Connect-gd-auth Wiring + _app.js Auth Gate

**Files:**
- Modify: `pages/_app.js`
- Modify: `gasket.js` (add express middleware)

- [ ] **Step 1: Add connect-gd-auth middleware to gasket.js**

`connect-gd-auth` is CJS-only, so we use `createRequire` at the top of `gasket.js` and register the middleware via a plugin hook. The hook must be `async` because Gasket 7's `middleware` lifecycle is async.

```js
// Add to the TOP of gasket.js (alongside other imports):
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
const gdAuth = require('connect-gd-auth');

// Add to the plugins array:
{
  name: 'cds-admin-auth',
  hooks: {
    async middleware() {
      return [
        gdAuth({
          appName: 'cds-admin',
          verify: gdAuth.risk.low,
          types: ['jomax'],
          auth: ['basic'],
          host: process.env.SSO_HOST || 'sso.int.test-gdcorp.tools'
        })
      ];
    }
  }
}
```

- [ ] **Step 2: Implement pages/_app.js auth gate**

```jsx
// pages/_app.js
// Relies on connect-gd-auth's Origin-Location or Referer handling on the server.
// When /auth/me returns 401, we just reload — the server-side auth middleware
// will redirect the browser to SSO (connect-gd-auth handles the redirect URL).
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

- [ ] `npm run local` — dev server starts, no startup errors
- [ ] `/` — component list loads from Switchboard dev env
- [ ] `/components/new` — create a test component, confirm 3 Switchboard entries created (cds, cds-auth, cds-admin)
- [ ] `/components/{name}` Manifest tab — Monaco loads, invalid JSON disables Save, valid JSON saves
- [ ] History tab — entries appear, Rollback reverts and updates manifest view
- [ ] Sync tab — prod→test copy shows in Switchboard
- [ ] i18n tab — set hash, confirm key created; rollback reverts
- [ ] RUM tab — set JSON config, confirm JSON validation works
- [ ] Access tab — add/remove jomax users, confirm cds-auth updated
- [ ] Alerts tab — enter webhook URL, test message fires to Slack
- [ ] Analytics page — select component, charts render (errors, volume, vitals)
- [ ] `npm test` — all tests pass
- [ ] Deploy to Katana dev — IAM writes succeed without cert env vars

---

## Revision Notes

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
