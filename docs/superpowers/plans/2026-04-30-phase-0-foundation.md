# OrderFlow Phase 0: Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the foundation for OrderFlow — monorepo, database, M365 SSO end-to-end, multi-tenant middleware, RBAC framework, and the UI shell — so Phase 1 can start adding feature code without re-doing infrastructure.

**Architecture:** Bun workspace monorepo with `apps/web` (React + Vite + Tailwind + shadcn) and `apps/api` (Cloudflare Workers + Hono + Drizzle ORM + D1). Auth via Microsoft Entra ID (OIDC PKCE) with HS256 session JWT in HttpOnly cookies. Multi-tenant isolation enforced by middleware + DB query helper. Localhost-only deployment in this phase.

**Tech Stack:** Bun, TypeScript, React 18, Vite, Tailwind, shadcn/ui, TanStack Query, TanStack Router, Hono, Drizzle ORM, Cloudflare D1 (Wrangler local), jose (JWT), Vitest, Playwright.

**Phase boundary:** This plan ends when a user can log in via M365, see an empty shell with their tenant name and role, and the audit log records the login. No business features (orders/customers) are implemented in this phase — those start in Phase 1.

---

## Reference Spec

Always cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §5 Data model (only `tenants`, `users`, `teams`, `sessions`, `audit_log` in this phase)
- §9 SSO flow
- §4 RBAC (framework only, full matrix wired in Phase 1)
- §13 Directory structure

---

## File Structure (created by this plan)

```
orderflow/
├── package.json                        # workspace root
├── bun.lockb
├── tsconfig.base.json
├── .gitignore
├── .editorconfig
├── apps/
│   ├── web/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── postcss.config.js
│   │   ├── index.html
│   │   ├── src/
│   │   │   ├── main.tsx
│   │   │   ├── App.tsx
│   │   │   ├── routes/
│   │   │   │   ├── __root.tsx
│   │   │   │   ├── index.tsx           # post-login dashboard placeholder
│   │   │   │   └── login.tsx
│   │   │   ├── components/
│   │   │   │   ├── ui/                 # shadcn
│   │   │   │   └── shell/
│   │   │   │       ├── AppShell.tsx
│   │   │   │       ├── Sidebar.tsx
│   │   │   │       └── TopBar.tsx
│   │   │   ├── lib/
│   │   │   │   ├── api.ts              # fetch helpers
│   │   │   │   └── auth.ts             # client-side auth helpers
│   │   │   ├── i18n/
│   │   │   │   ├── index.ts
│   │   │   │   ├── zh-TW.json
│   │   │   │   └── en.json
│   │   │   └── styles/
│   │   │       └── globals.css
│   │   └── tests/
│   │       └── e2e/
│   │           └── auth.spec.ts        # Playwright
│   └── api/
│       ├── package.json
│       ├── tsconfig.json
│       ├── wrangler.toml
│       ├── drizzle.config.ts
│       ├── src/
│       │   ├── index.ts                # Hono app entry
│       │   ├── env.ts                  # bindings + secrets types
│       │   ├── routes/
│       │   │   ├── auth.ts             # /auth/login /auth/callback /auth/logout /auth/me
│       │   │   └── health.ts
│       │   ├── auth/
│       │   │   ├── entra.ts            # OIDC discovery + token exchange
│       │   │   ├── session.ts          # JWT sign/verify
│       │   │   ├── middleware.ts       # auth + tenant context
│       │   │   └── rbac.ts             # role helpers (framework)
│       │   ├── db/
│       │   │   ├── schema.ts           # Drizzle schema
│       │   │   ├── client.ts           # getDb(tenantId) helper
│       │   │   └── audit.ts            # audit_log writer
│       │   ├── lib/
│       │   │   ├── ulid.ts             # ULID generator
│       │   │   ├── crypto.ts           # PKCE helpers, HMAC
│       │   │   └── errors.ts           # AppError + status mapper
│       │   └── seed.ts                 # dev seed script
│       ├── migrations/                 # drizzle-kit output
│       └── tests/
│           ├── auth/
│           │   ├── session.test.ts
│           │   ├── entra.test.ts
│           │   └── middleware.test.ts
│           ├── db/
│           │   ├── client.test.ts
│           │   └── audit.test.ts
│           └── lib/
│               └── crypto.test.ts
└── docs/                               # already exists
```

---

## Task 1: Monorepo Skeleton

**Files:**
- Create: `orderflow/package.json`
- Create: `orderflow/tsconfig.base.json`
- Create: `orderflow/.gitignore`
- Create: `orderflow/.editorconfig`

- [ ] **Step 1: Verify project root state**

```bash
cd /Users/swryociao/orderflow
git status
ls -la
```

Expected: only `docs/` directory and prior commits exist. Working tree clean.

- [ ] **Step 2: Create workspace `package.json`**

```json
{
  "name": "orderflow",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev:web": "bun --cwd apps/web run dev",
    "dev:api": "bun --cwd apps/api run dev",
    "test": "bun --cwd apps/api run test && bun --cwd apps/web run test",
    "typecheck": "bun --cwd apps/web run typecheck && bun --cwd apps/api run typecheck"
  },
  "engines": {
    "bun": ">=1.1.0"
  }
}
```

- [ ] **Step 3: Create `tsconfig.base.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": false,
    "resolveJsonModule": true,
    "lib": ["ES2022"]
  }
}
```

- [ ] **Step 4: Create `.gitignore`**

```
node_modules/
dist/
.wrangler/
*.log
.env
.env.local
.DS_Store
.turbo/
coverage/
test-results/
playwright-report/
```

- [ ] **Step 5: Create `.editorconfig`**

```
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

- [ ] **Step 6: Commit**

```bash
git add package.json tsconfig.base.json .gitignore .editorconfig
git commit -m "chore: monorepo skeleton (package.json + tsconfig base + gitignore)"
```

---

## Task 2: Web App Scaffold (Vite + React + Tailwind)

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/tsconfig.json`
- Create: `apps/web/vite.config.ts`
- Create: `apps/web/tailwind.config.ts`
- Create: `apps/web/postcss.config.js`
- Create: `apps/web/index.html`
- Create: `apps/web/src/main.tsx`
- Create: `apps/web/src/App.tsx`
- Create: `apps/web/src/styles/globals.css`

- [ ] **Step 1: Create `apps/web/package.json`**

```json
{
  "name": "@orderflow/web",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite --port 5173",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:e2e": "playwright test",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@tanstack/react-query": "^5.51.0",
    "@tanstack/react-router": "^1.45.0",
    "i18next": "^23.11.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-i18next": "^14.1.0"
  },
  "devDependencies": {
    "@playwright/test": "^1.45.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38",
    "tailwindcss": "^3.4.4",
    "typescript": "^5.5.0",
    "vite": "^5.3.0",
    "vitest": "^1.6.0"
  }
}
```

- [ ] **Step 2: Create `apps/web/tsconfig.json`**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": false,
    "noEmit": true,
    "types": ["vite/client"],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src", "tests"]
}
```

- [ ] **Step 3: Create `apps/web/vite.config.ts`**

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    proxy: {
      '/api': 'http://localhost:8787',
      '/auth': 'http://localhost:8787',
    },
  },
});
```

- [ ] **Step 4: Create `apps/web/tailwind.config.ts`**

```ts
import type { Config } from 'tailwindcss';

export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: { extend: {} },
  plugins: [],
} satisfies Config;
```

- [ ] **Step 5: Create `apps/web/postcss.config.js`**

```js
export default {
  plugins: { tailwindcss: {}, autoprefixer: {} },
};
```

- [ ] **Step 6: Create `apps/web/index.html`**

```html
<!doctype html>
<html lang="zh-TW">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
    <title>OrderFlow</title>
  </head>
  <body class="bg-slate-50 text-slate-900">
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 7: Create `apps/web/src/styles/globals.css`**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

html, body, #root { height: 100%; }
```

- [ ] **Step 8: Create `apps/web/src/main.tsx`**

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App';
import './styles/globals.css';

const qc = new QueryClient();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={qc}>
      <App />
    </QueryClientProvider>
  </React.StrictMode>,
);
```

- [ ] **Step 9: Create `apps/web/src/App.tsx`**

```tsx
export default function App() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">OrderFlow — scaffold</h1>
      <p className="text-slate-600">If you can read this, the web app is alive.</p>
    </main>
  );
}
```

- [ ] **Step 10: Install deps and run dev server**

```bash
cd /Users/swryociao/orderflow
bun install
bun --cwd apps/web run dev
```

Expected: dev server boots on `http://localhost:5173` with the heading "OrderFlow — scaffold". Stop with Ctrl+C.

- [ ] **Step 11: Commit**

```bash
git add apps/web bun.lockb package.json
git commit -m "feat(web): scaffold Vite + React + Tailwind"
```

---

## Task 3: API App Scaffold (Workers + Hono)

**Files:**
- Create: `apps/api/package.json`
- Create: `apps/api/tsconfig.json`
- Create: `apps/api/wrangler.toml`
- Create: `apps/api/src/env.ts`
- Create: `apps/api/src/index.ts`
- Create: `apps/api/src/routes/health.ts`
- Create: `apps/api/src/lib/errors.ts`
- Create: `apps/api/tests/setup.ts`
- Create: `apps/api/vitest.config.ts`

- [ ] **Step 1: Create `apps/api/package.json`**

```json
{
  "name": "@orderflow/api",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "wrangler dev --port 8787",
    "deploy": "wrangler deploy",
    "test": "vitest run",
    "typecheck": "tsc --noEmit",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "wrangler d1 migrations apply orderflow_dev --local",
    "db:seed": "wrangler dev --test-scheduled --port 8788 src/seed.ts"
  },
  "dependencies": {
    "drizzle-orm": "^0.32.0",
    "hono": "^4.5.0",
    "jose": "^5.4.0",
    "ulid": "^2.3.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@cloudflare/vitest-pool-workers": "^0.4.0",
    "@cloudflare/workers-types": "^4.20240712.0",
    "drizzle-kit": "^0.24.0",
    "typescript": "^5.5.0",
    "vitest": "^1.6.0",
    "wrangler": "^3.65.0"
  }
}
```

- [ ] **Step 2: Create `apps/api/tsconfig.json`**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "moduleResolution": "bundler",
    "types": ["@cloudflare/workers-types"],
    "noEmit": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", "tests"]
}
```

- [ ] **Step 3: Create `apps/api/wrangler.toml`**

```toml
name = "orderflow-api"
main = "src/index.ts"
compatibility_date = "2026-04-01"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "orderflow_dev"
database_id = "00000000-0000-0000-0000-000000000000"   # replaced by wrangler d1 create later
migrations_dir = "migrations"

[[kv_namespaces]]
binding = "KV"
id = "00000000000000000000000000000000"

[vars]
ENVIRONMENT = "dev"
ENTRA_AUTHORITY = "https://login.microsoftonline.com/common/v2.0"
APP_BASE_URL = "http://localhost:5173"
API_BASE_URL = "http://localhost:8787"
SESSION_COOKIE_NAME = "of_session"

# Secrets set via `wrangler secret put` (NOT committed):
#   ENTRA_CLIENT_ID
#   ENTRA_CLIENT_SECRET
#   SESSION_JWT_SECRET   (32+ random bytes, base64)
```

- [ ] **Step 4: Create `apps/api/src/env.ts`**

```ts
export interface Env {
  DB: D1Database;
  KV: KVNamespace;
  ENVIRONMENT: 'dev' | 'staging' | 'prod';
  ENTRA_AUTHORITY: string;
  ENTRA_CLIENT_ID: string;
  ENTRA_CLIENT_SECRET: string;
  APP_BASE_URL: string;
  API_BASE_URL: string;
  SESSION_COOKIE_NAME: string;
  SESSION_JWT_SECRET: string;
}

export type AppCtx = {
  Bindings: Env;
  Variables: {
    userId?: string;
    tenantId?: string;
    role?: string;
  };
};
```

- [ ] **Step 5: Create `apps/api/src/lib/errors.ts`**

```ts
export class AppError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
    public details?: unknown,
  ) {
    super(message);
  }
}

export const Errors = {
  Unauthorized: () => new AppError(401, 'unauthorized', 'Authentication required'),
  Forbidden: (reason = 'Insufficient permissions') => new AppError(403, 'forbidden', reason),
  NotFound: (what = 'Resource') => new AppError(404, 'not_found', `${what} not found`),
  BadRequest: (msg: string, details?: unknown) => new AppError(400, 'bad_request', msg, details),
  Conflict: (msg: string) => new AppError(409, 'conflict', msg),
};
```

- [ ] **Step 6: Create `apps/api/src/routes/health.ts`**

```ts
import { Hono } from 'hono';
import type { AppCtx } from '../env';

export const health = new Hono<AppCtx>();

health.get('/', (c) =>
  c.json({ ok: true, env: c.env.ENVIRONMENT, ts: new Date().toISOString() }),
);
```

- [ ] **Step 7: Create `apps/api/src/index.ts`**

```ts
import { Hono } from 'hono';
import type { AppCtx } from './env';
import { health } from './routes/health';
import { AppError } from './lib/errors';

const app = new Hono<AppCtx>();

app.onError((err, c) => {
  if (err instanceof AppError) {
    return c.json({ error: err.code, message: err.message, details: err.details }, err.status);
  }
  console.error('unhandled', err);
  return c.json({ error: 'internal_error', message: 'Internal server error' }, 500);
});

app.route('/api/health', health);

export default app;
```

- [ ] **Step 8: Create `apps/api/vitest.config.ts`**

```ts
import { defineWorkersConfig } from '@cloudflare/vitest-pool-workers/config';

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: './wrangler.toml' },
        miniflare: {
          d1Databases: ['DB'],
          kvNamespaces: ['KV'],
          bindings: {
            ENVIRONMENT: 'dev',
            ENTRA_AUTHORITY: 'https://login.microsoftonline.com/common/v2.0',
            ENTRA_CLIENT_ID: 'test-client',
            ENTRA_CLIENT_SECRET: 'test-secret',
            APP_BASE_URL: 'http://localhost:5173',
            API_BASE_URL: 'http://localhost:8787',
            SESSION_COOKIE_NAME: 'of_session',
            SESSION_JWT_SECRET: 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa',
          },
        },
      },
    },
  },
});
```

- [ ] **Step 9: Run dev server to verify boot**

```bash
cd /Users/swryociao/orderflow
bun install
bun --cwd apps/api run dev
```

Expected: `wrangler dev` boots on `http://localhost:8787`. In another terminal: `curl http://localhost:8787/api/health` → returns `{"ok":true,"env":"dev",...}`. Stop with Ctrl+C.

- [ ] **Step 10: Commit**

```bash
git add apps/api bun.lockb
git commit -m "feat(api): scaffold Cloudflare Worker + Hono with health route"
```

---

## Task 4: D1 Database + Drizzle Setup

**Files:**
- Create: `apps/api/drizzle.config.ts`
- Create: `apps/api/src/db/schema.ts`
- Create: `apps/api/src/db/client.ts`
- Modify: `apps/api/wrangler.toml` (replace `database_id` after `d1 create`)

- [ ] **Step 1: Create local D1 database**

```bash
cd /Users/swryociao/orderflow
bun --cwd apps/api wrangler d1 create orderflow_dev
```

Expected: outputs a `database_id`. Copy it and paste into `apps/api/wrangler.toml` replacing the placeholder `00000000-0000-0000-0000-000000000000`.

- [ ] **Step 2: Create `apps/api/drizzle.config.ts`**

```ts
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
  // d1-http config not needed for `generate`; for `push`/`studio` read .dev.vars
} satisfies Config;
```

- [ ] **Step 3: Create `apps/api/src/db/schema.ts` (Phase 0 tables only)**

```ts
import { sqliteTable, text, integer, index, uniqueIndex } from 'drizzle-orm/sqlite-core';

export const tenants = sqliteTable('tenants', {
  id: text('id').primaryKey(),                     // ULID
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  plan: text('plan').notNull().default('free'),
  entraTenantId: text('entra_tenant_id'),
  highValueThreshold: integer('high_value_threshold').notNull().default(1000000),
  localeDefault: text('locale_default').notNull().default('zh-TW'),
  createdAt: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
});

export const teams = sqliteTable('teams', {
  id: text('id').primaryKey(),
  tenantId: text('tenant_id').notNull().references(() => tenants.id),
  name: text('name').notNull(),
  managerUserId: text('manager_user_id'),          // FK added later (avoids circular)
  createdAt: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
}, (t) => ({
  tenantIdx: index('teams_tenant_idx').on(t.tenantId),
}));

export const users = sqliteTable('users', {
  id: text('id').primaryKey(),
  tenantId: text('tenant_id').notNull().references(() => tenants.id),
  email: text('email').notNull(),
  displayName: text('display_name').notNull(),
  entraOid: text('entra_oid'),
  role: text('role').notNull().default('viewer'),  // admin/sales_manager/sales/ops/accountant/viewer
  teamId: text('team_id').references(() => teams.id),
  status: text('status').notNull().default('active'), // active/suspended/invited
  teamsUserId: text('teams_user_id'),
  slackUserId: text('slack_user_id'),
  createdAt: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
}, (t) => ({
  tenantEmailIdx: uniqueIndex('users_tenant_email_idx').on(t.tenantId, t.email),
  tenantOidIdx: uniqueIndex('users_tenant_oid_idx').on(t.tenantId, t.entraOid),
}));

export const sessions = sqliteTable('sessions', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id),
  tenantId: text('tenant_id').notNull().references(() => tenants.id),
  deviceLabel: text('device_label'),
  expiresAt: integer('expires_at', { mode: 'timestamp_ms' }).notNull(),
  lastActiveAt: integer('last_active_at', { mode: 'timestamp_ms' }).notNull(),
  ip: text('ip'),
  userAgent: text('user_agent'),
}, (t) => ({
  userIdx: index('sessions_user_idx').on(t.userId),
}));

export const auditLog = sqliteTable('audit_log', {
  id: text('id').primaryKey(),
  tenantId: text('tenant_id').notNull().references(() => tenants.id),
  actorUserId: text('actor_user_id'),
  actedAsUserId: text('acted_as_user_id'),
  via: text('via').notNull().default('web'),       // web/mobile/mcp/system
  action: text('action').notNull(),                // e.g. 'auth.login', 'order.create'
  targetType: text('target_type'),
  targetId: text('target_id'),
  before: text('before'),                          // JSON string
  after: text('after'),                            // JSON string
  ip: text('ip'),
  userAgent: text('user_agent'),
  createdAt: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
}, (t) => ({
  tenantTimeIdx: index('audit_tenant_time_idx').on(t.tenantId, t.createdAt),
}));
```

- [ ] **Step 4: Create `apps/api/src/db/client.ts`**

```ts
import { drizzle } from 'drizzle-orm/d1';
import * as schema from './schema';

export type DbClient = ReturnType<typeof drizzle<typeof schema>>;

/**
 * Returns a Drizzle client. The tenantId is **not** baked into the connection
 * (D1 has no row-level security); instead, we expose this helper and require
 * every business query to filter by `tenantId` explicitly. The auth middleware
 * sets `c.var.tenantId` and route handlers MUST use it.
 */
export function getDb(d1: D1Database): DbClient {
  return drizzle(d1, { schema });
}
```

- [ ] **Step 5: Generate first migration**

```bash
cd /Users/swryociao/orderflow
bun --cwd apps/api run db:generate
```

Expected: file appears at `apps/api/migrations/0000_*.sql`.

- [ ] **Step 6: Apply migration to local D1**

```bash
bun --cwd apps/api run db:migrate
```

Expected: `Migrations applied`.

- [ ] **Step 7: Sanity check via wrangler**

```bash
bun --cwd apps/api wrangler d1 execute orderflow_dev --local --command "SELECT name FROM sqlite_master WHERE type='table';"
```

Expected: lists `tenants`, `users`, `teams`, `sessions`, `audit_log`, plus drizzle's `__drizzle_migrations`.

- [ ] **Step 8: Commit**

```bash
git add apps/api/drizzle.config.ts apps/api/src/db apps/api/migrations apps/api/wrangler.toml
git commit -m "feat(api): D1 schema for tenants/users/teams/sessions/audit_log"
```

---

## Task 5: ULID + Crypto Helpers (with tests)

**Files:**
- Create: `apps/api/src/lib/ulid.ts`
- Create: `apps/api/src/lib/crypto.ts`
- Create: `apps/api/tests/lib/crypto.test.ts`

- [ ] **Step 1: Write failing test for crypto helpers**

Create `apps/api/tests/lib/crypto.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { generatePkce, hmacShort, randomBase64Url } from '../../src/lib/crypto';

describe('crypto', () => {
  it('generates a PKCE pair where verifier has 43-128 chars and challenge derives from it', async () => {
    const { verifier, challenge, method } = await generatePkce();
    expect(verifier.length).toBeGreaterThanOrEqual(43);
    expect(verifier.length).toBeLessThanOrEqual(128);
    expect(method).toBe('S256');
    expect(challenge).toMatch(/^[A-Za-z0-9_-]+$/); // base64url
  });

  it('hmacShort returns deterministic 8-char base64url for same input/key', async () => {
    const a = await hmacShort('order-123', 'secret');
    const b = await hmacShort('order-123', 'secret');
    expect(a).toBe(b);
    expect(a.length).toBe(8);
    expect(a).toMatch(/^[A-Za-z0-9_-]+$/);
  });

  it('hmacShort differs for different inputs', async () => {
    const a = await hmacShort('order-123', 'secret');
    const b = await hmacShort('order-124', 'secret');
    expect(a).not.toBe(b);
  });

  it('randomBase64Url produces requested byte length encoded url-safe', () => {
    const s = randomBase64Url(32);
    expect(s).toMatch(/^[A-Za-z0-9_-]+$/);
    expect(s.length).toBe(43); // 32 bytes → 43 chars base64url no pad
  });
});
```

- [ ] **Step 2: Run test to confirm failure**

```bash
bun --cwd apps/api run test -- crypto.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `apps/api/src/lib/crypto.ts`**

```ts
function toBase64Url(buf: ArrayBuffer | Uint8Array): string {
  const bytes = buf instanceof Uint8Array ? buf : new Uint8Array(buf);
  let str = '';
  for (let i = 0; i < bytes.byteLength; i++) str += String.fromCharCode(bytes[i]!);
  return btoa(str).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

export function randomBase64Url(byteLen: number): string {
  const bytes = new Uint8Array(byteLen);
  crypto.getRandomValues(bytes);
  return toBase64Url(bytes);
}

export async function generatePkce(): Promise<{
  verifier: string;
  challenge: string;
  method: 'S256';
}> {
  const verifier = randomBase64Url(64); // ~86 chars
  const enc = new TextEncoder().encode(verifier);
  const digest = await crypto.subtle.digest('SHA-256', enc);
  return { verifier, challenge: toBase64Url(digest), method: 'S256' };
}

export async function hmacShort(message: string, secret: string): Promise<string> {
  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign'],
  );
  const sig = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(message));
  return toBase64Url(sig).slice(0, 8);
}
```

- [ ] **Step 4: Implement `apps/api/src/lib/ulid.ts`**

```ts
import { ulid as makeUlid } from 'ulid';

export function newId(): string {
  return makeUlid();
}
```

- [ ] **Step 5: Re-run test to verify pass**

```bash
bun --cwd apps/api run test -- crypto.test.ts
```

Expected: 4/4 PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/lib apps/api/tests/lib
git commit -m "feat(api): crypto helpers (PKCE, HMAC short, base64url) + ULID wrapper"
```

---

## Task 6: Session JWT (sign / verify)

**Files:**
- Create: `apps/api/src/auth/session.ts`
- Create: `apps/api/tests/auth/session.test.ts`

- [ ] **Step 1: Write failing tests**

Create `apps/api/tests/auth/session.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { signSession, verifySession, type SessionPayload } from '../../src/auth/session';

const SECRET = 'a'.repeat(32);

describe('session jwt', () => {
  it('signs and verifies a valid token', async () => {
    const payload: SessionPayload = {
      sub: 'user_01',
      tid: 'tenant_01',
      role: 'sales',
      sid: 'sess_01',
    };
    const token = await signSession(payload, SECRET, 60);
    const decoded = await verifySession(token, SECRET);
    expect(decoded.sub).toBe('user_01');
    expect(decoded.tid).toBe('tenant_01');
    expect(decoded.role).toBe('sales');
    expect(decoded.sid).toBe('sess_01');
  });

  it('rejects an expired token', async () => {
    const token = await signSession({ sub: 'u', tid: 't', role: 'viewer', sid: 's' }, SECRET, -1);
    await expect(verifySession(token, SECRET)).rejects.toThrow();
  });

  it('rejects a token signed with a different secret', async () => {
    const token = await signSession({ sub: 'u', tid: 't', role: 'viewer', sid: 's' }, SECRET, 60);
    await expect(verifySession(token, 'b'.repeat(32))).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Run to confirm failure**

```bash
bun --cwd apps/api run test -- session.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `apps/api/src/auth/session.ts`**

```ts
import { SignJWT, jwtVerify } from 'jose';

export interface SessionPayload {
  sub: string;        // user id
  tid: string;        // tenant id
  role: string;
  sid: string;        // session row id (for revocation)
}

export async function signSession(
  payload: SessionPayload,
  secret: string,
  ttlSeconds: number,
): Promise<string> {
  const key = new TextEncoder().encode(secret);
  return await new SignJWT({ ...payload })
    .setProtectedHeader({ alg: 'HS256', typ: 'JWT' })
    .setIssuedAt()
    .setExpirationTime(Math.floor(Date.now() / 1000) + ttlSeconds)
    .setIssuer('orderflow')
    .setAudience('orderflow-web')
    .sign(key);
}

export async function verifySession(token: string, secret: string): Promise<SessionPayload> {
  const key = new TextEncoder().encode(secret);
  const { payload } = await jwtVerify(token, key, {
    issuer: 'orderflow',
    audience: 'orderflow-web',
  });
  const { sub, tid, role, sid } = payload as Record<string, unknown>;
  if (typeof sub !== 'string' || typeof tid !== 'string' || typeof role !== 'string' || typeof sid !== 'string') {
    throw new Error('invalid session payload shape');
  }
  return { sub, tid, role, sid };
}
```

- [ ] **Step 4: Re-run tests to verify pass**

```bash
bun --cwd apps/api run test -- session.test.ts
```

Expected: 3/3 PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/auth/session.ts apps/api/tests/auth/session.test.ts
git commit -m "feat(api): session JWT sign/verify with HS256 + issuer/audience claims"
```

---

## Task 7: Entra OIDC Helpers (discovery + token exchange + ID token verify)

**Files:**
- Create: `apps/api/src/auth/entra.ts`
- Create: `apps/api/tests/auth/entra.test.ts`

- [ ] **Step 1: Write failing tests (using fetch mocking)**

Create `apps/api/tests/auth/entra.test.ts`:

```ts
import { afterEach, describe, expect, it, vi } from 'vitest';
import { exchangeCodeForTokens, parseIdToken } from '../../src/auth/entra';

const ORIGINAL_FETCH = globalThis.fetch;

afterEach(() => {
  globalThis.fetch = ORIGINAL_FETCH;
  vi.restoreAllMocks();
});

describe('entra', () => {
  it('exchangeCodeForTokens posts to authority with PKCE verifier and parses tokens', async () => {
    const fakeTokens = { access_token: 'a', id_token: 'header.payload.sig', expires_in: 3600 };
    globalThis.fetch = vi.fn(async (input: any, init: any) => {
      expect(String(input)).toContain('/oauth2/v2.0/token');
      const body = new URLSearchParams(init.body);
      expect(body.get('grant_type')).toBe('authorization_code');
      expect(body.get('code')).toBe('CODE');
      expect(body.get('code_verifier')).toBe('VERIFIER');
      expect(body.get('client_id')).toBe('CID');
      return new Response(JSON.stringify(fakeTokens), { status: 200 });
    }) as any;

    const out = await exchangeCodeForTokens({
      authority: 'https://login.microsoftonline.com/common/v2.0',
      clientId: 'CID',
      clientSecret: 'CS',
      code: 'CODE',
      redirectUri: 'http://localhost:8787/auth/callback',
      codeVerifier: 'VERIFIER',
    });
    expect(out.id_token).toBe('header.payload.sig');
  });

  it('parseIdToken extracts oid/email/tid/name from a known JWT payload', () => {
    const payload = {
      oid: 'oid-123',
      email: 'user@example.com',
      preferred_username: 'user@example.com',
      name: 'Test User',
      tid: 'tenant-abc',
    };
    const b64 = (o: unknown) =>
      btoa(JSON.stringify(o)).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
    const fake = `${b64({ alg: 'RS256' })}.${b64(payload)}.sig`;
    const out = parseIdToken(fake);
    expect(out.oid).toBe('oid-123');
    expect(out.email).toBe('user@example.com');
    expect(out.tid).toBe('tenant-abc');
    expect(out.name).toBe('Test User');
  });
});
```

> Note: full JWKS verification of the id_token signature is **NOT** done in Phase 0 (deferred to Phase 1 hardening). For dev we trust the token returned by the same TLS-secured POST that authenticated us. We document this as a known gap.

- [ ] **Step 2: Run to confirm failure**

```bash
bun --cwd apps/api run test -- entra.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `apps/api/src/auth/entra.ts`**

```ts
export interface TokenResponse {
  access_token: string;
  id_token: string;
  expires_in: number;
  refresh_token?: string;
}

export interface IdTokenClaims {
  oid: string;
  email: string;
  tid: string;
  name: string;
}

export async function exchangeCodeForTokens(input: {
  authority: string;
  clientId: string;
  clientSecret: string;
  code: string;
  redirectUri: string;
  codeVerifier: string;
}): Promise<TokenResponse> {
  const tokenUrl = `${input.authority.replace(/\/$/, '')}/oauth2/v2.0/token`;
  const body = new URLSearchParams({
    grant_type: 'authorization_code',
    client_id: input.clientId,
    client_secret: input.clientSecret,
    code: input.code,
    redirect_uri: input.redirectUri,
    code_verifier: input.codeVerifier,
    scope: 'openid profile email offline_access',
  });
  const res = await fetch(tokenUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: body.toString(),
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`Entra token exchange failed: ${res.status} ${text}`);
  }
  return (await res.json()) as TokenResponse;
}

export function parseIdToken(idToken: string): IdTokenClaims {
  const parts = idToken.split('.');
  if (parts.length < 2) throw new Error('invalid id_token');
  const b64 = parts[1]!.replace(/-/g, '+').replace(/_/g, '/');
  const padded = b64 + '='.repeat((4 - (b64.length % 4)) % 4);
  const json = atob(padded);
  const claims = JSON.parse(json) as Record<string, unknown>;
  const oid = claims.oid;
  const email = claims.email ?? claims.preferred_username;
  const tid = claims.tid;
  const name = claims.name;
  if (typeof oid !== 'string' || typeof email !== 'string' || typeof tid !== 'string' || typeof name !== 'string') {
    throw new Error('id_token missing required claims');
  }
  return { oid, email, tid, name };
}
```

- [ ] **Step 4: Re-run tests**

```bash
bun --cwd apps/api run test -- entra.test.ts
```

Expected: 2/2 PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/auth/entra.ts apps/api/tests/auth/entra.test.ts
git commit -m "feat(api): Entra OIDC code exchange + id_token claim parsing (no JWKS yet)"
```

---

## Task 8: Audit Log Writer

**Files:**
- Create: `apps/api/src/db/audit.ts`
- Create: `apps/api/tests/db/audit.test.ts`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/db/audit.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { env } from 'cloudflare:test';
import { drizzle } from 'drizzle-orm/d1';
import { eq } from 'drizzle-orm';
import { writeAudit } from '../../src/db/audit';
import * as schema from '../../src/db/schema';

describe('writeAudit', () => {
  it('inserts an audit row with provided fields', async () => {
    const db = drizzle(env.DB, { schema });
    // pre-seed a tenant row that audit_log FK points to
    await db.insert(schema.tenants).values({
      id: 'tenant_test',
      name: 'Test',
      slug: 'test',
      plan: 'free',
      highValueThreshold: 1000000,
      localeDefault: 'zh-TW',
      createdAt: new Date(),
    });
    await writeAudit(db, {
      tenantId: 'tenant_test',
      actorUserId: 'user_test',
      via: 'web',
      action: 'auth.login',
      ip: '127.0.0.1',
      userAgent: 'vitest',
    });
    const rows = await db.select().from(schema.auditLog).where(eq(schema.auditLog.tenantId, 'tenant_test'));
    expect(rows).toHaveLength(1);
    expect(rows[0]!.action).toBe('auth.login');
    expect(rows[0]!.via).toBe('web');
  });
});
```

- [ ] **Step 2: Run to confirm failure**

```bash
bun --cwd apps/api run test -- audit.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `apps/api/src/db/audit.ts`**

```ts
import type { DbClient } from './client';
import { auditLog } from './schema';
import { newId } from '../lib/ulid';

export interface AuditInput {
  tenantId: string;
  actorUserId?: string;
  actedAsUserId?: string;
  via?: 'web' | 'mobile' | 'mcp' | 'system';
  action: string;
  targetType?: string;
  targetId?: string;
  before?: unknown;
  after?: unknown;
  ip?: string;
  userAgent?: string;
}

export async function writeAudit(db: DbClient, input: AuditInput): Promise<void> {
  await db.insert(auditLog).values({
    id: newId(),
    tenantId: input.tenantId,
    actorUserId: input.actorUserId ?? null,
    actedAsUserId: input.actedAsUserId ?? null,
    via: input.via ?? 'web',
    action: input.action,
    targetType: input.targetType ?? null,
    targetId: input.targetId ?? null,
    before: input.before === undefined ? null : JSON.stringify(input.before),
    after: input.after === undefined ? null : JSON.stringify(input.after),
    ip: input.ip ?? null,
    userAgent: input.userAgent ?? null,
    createdAt: new Date(),
  });
}
```

- [ ] **Step 4: Re-run test**

```bash
bun --cwd apps/api run test -- audit.test.ts
```

Expected: 1/1 PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/db/audit.ts apps/api/tests/db/audit.test.ts
git commit -m "feat(api): audit log writer with JSON before/after serialization"
```

---

## Task 9: Auth Middleware (cookie → session → context)

**Files:**
- Create: `apps/api/src/auth/middleware.ts`
- Create: `apps/api/tests/auth/middleware.test.ts`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/auth/middleware.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { Hono } from 'hono';
import { env } from 'cloudflare:test';
import type { AppCtx } from '../../src/env';
import { requireAuth } from '../../src/auth/middleware';
import { signSession } from '../../src/auth/session';

function buildApp() {
  const app = new Hono<AppCtx>();
  app.use('*', requireAuth);
  app.get('/me', (c) =>
    c.json({ userId: c.var.userId, tenantId: c.var.tenantId, role: c.var.role }),
  );
  return app;
}

describe('requireAuth middleware', () => {
  it('returns 401 when no cookie', async () => {
    const res = await buildApp().request('/me', {}, env);
    expect(res.status).toBe(401);
  });

  it('attaches userId/tenantId/role on valid cookie', async () => {
    const token = await signSession(
      { sub: 'u1', tid: 't1', role: 'admin', sid: 's1' },
      env.SESSION_JWT_SECRET,
      60,
    );
    const res = await buildApp().request(
      '/me',
      { headers: { cookie: `${env.SESSION_COOKIE_NAME}=${token}` } },
      env,
    );
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ userId: 'u1', tenantId: 't1', role: 'admin' });
  });

  it('returns 401 on tampered cookie', async () => {
    const res = await buildApp().request(
      '/me',
      { headers: { cookie: `${env.SESSION_COOKIE_NAME}=garbage` } },
      env,
    );
    expect(res.status).toBe(401);
  });
});
```

- [ ] **Step 2: Run to confirm failure**

```bash
bun --cwd apps/api run test -- middleware.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `apps/api/src/auth/middleware.ts`**

```ts
import type { MiddlewareHandler } from 'hono';
import type { AppCtx } from '../env';
import { verifySession } from './session';
import { Errors } from '../lib/errors';

function readCookie(header: string | null, name: string): string | null {
  if (!header) return null;
  const parts = header.split(/;\s*/);
  for (const p of parts) {
    const eq = p.indexOf('=');
    if (eq < 0) continue;
    if (p.slice(0, eq) === name) return decodeURIComponent(p.slice(eq + 1));
  }
  return null;
}

export const requireAuth: MiddlewareHandler<AppCtx> = async (c, next) => {
  const token = readCookie(c.req.header('cookie') ?? null, c.env.SESSION_COOKIE_NAME);
  if (!token) throw Errors.Unauthorized();
  let payload;
  try {
    payload = await verifySession(token, c.env.SESSION_JWT_SECRET);
  } catch {
    throw Errors.Unauthorized();
  }
  c.set('userId', payload.sub);
  c.set('tenantId', payload.tid);
  c.set('role', payload.role);
  await next();
};
```

- [ ] **Step 4: Re-run test**

```bash
bun --cwd apps/api run test -- middleware.test.ts
```

Expected: 3/3 PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/auth/middleware.ts apps/api/tests/auth/middleware.test.ts
git commit -m "feat(api): requireAuth middleware reads session cookie + populates ctx"
```

---

## Task 10: RBAC Helpers (framework, full matrix in Phase 1)

**Files:**
- Create: `apps/api/src/auth/rbac.ts`
- Create: `apps/api/tests/auth/rbac.test.ts`

- [ ] **Step 1: Write failing test**

Create `apps/api/tests/auth/rbac.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { hasRole, requireRole, ROLES, type Role } from '../../src/auth/rbac';
import { Errors } from '../../src/lib/errors';

describe('rbac', () => {
  it('hasRole returns true when role is in allowed list', () => {
    expect(hasRole('admin', ['admin', 'sales_manager'])).toBe(true);
    expect(hasRole('sales', ['admin'])).toBe(false);
  });

  it('requireRole throws Forbidden when role is not allowed', () => {
    expect(() => requireRole('viewer', ['admin'])).toThrowError(/forbidden/i);
  });

  it('requireRole returns void on allowed role', () => {
    expect(requireRole('admin', ['admin'])).toBeUndefined();
  });

  it('ROLES list matches the spec (6 roles)', () => {
    expect(ROLES).toEqual([
      'admin',
      'sales_manager',
      'sales',
      'ops',
      'accountant',
      'viewer',
    ] satisfies Role[]);
  });
});
```

- [ ] **Step 2: Run to confirm failure**

```bash
bun --cwd apps/api run test -- rbac.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `apps/api/src/auth/rbac.ts`**

```ts
import { Errors } from '../lib/errors';

export const ROLES = ['admin', 'sales_manager', 'sales', 'ops', 'accountant', 'viewer'] as const;
export type Role = typeof ROLES[number];

export function isRole(x: unknown): x is Role {
  return typeof x === 'string' && (ROLES as readonly string[]).includes(x);
}

export function hasRole(role: string, allowed: readonly Role[]): boolean {
  return (allowed as readonly string[]).includes(role);
}

export function requireRole(role: string, allowed: readonly Role[]): void {
  if (!hasRole(role, allowed)) {
    throw Errors.Forbidden(`role '${role}' not in [${allowed.join(', ')}]`);
  }
}
```

- [ ] **Step 4: Re-run test**

```bash
bun --cwd apps/api run test -- rbac.test.ts
```

Expected: 4/4 PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/auth/rbac.ts apps/api/tests/auth/rbac.test.ts
git commit -m "feat(api): rbac role list + hasRole/requireRole helpers (matrix wired in Phase 1)"
```

---

## Task 11: Auth Routes (login redirect / callback / me / logout)

**Files:**
- Create: `apps/api/src/routes/auth.ts`
- Modify: `apps/api/src/index.ts` (mount /auth and /api/me)
- Create: `apps/api/tests/auth/routes.test.ts`

- [ ] **Step 1: Write failing test for /auth/login redirect**

Create `apps/api/tests/auth/routes.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { env } from 'cloudflare:test';
import app from '../../src/index';

describe('GET /auth/login', () => {
  it('returns a 302 redirect to Microsoft authorize endpoint', async () => {
    const res = await app.request('/auth/login', {}, env);
    expect(res.status).toBe(302);
    const loc = res.headers.get('location') ?? '';
    expect(loc).toContain('login.microsoftonline.com');
    expect(loc).toContain('client_id=');
    expect(loc).toContain('code_challenge=');
    expect(loc).toContain('code_challenge_method=S256');
    expect(loc).toContain('response_type=code');
  });
});

describe('GET /api/me without cookie', () => {
  it('returns 401', async () => {
    const res = await app.request('/api/me', {}, env);
    expect(res.status).toBe(401);
  });
});
```

- [ ] **Step 2: Run to confirm failure**

```bash
bun --cwd apps/api run test -- routes.test.ts
```

Expected: FAIL — `/auth/login` 404.

- [ ] **Step 3: Implement `apps/api/src/routes/auth.ts`**

```ts
import { Hono } from 'hono';
import { eq, and } from 'drizzle-orm';
import type { AppCtx } from '../env';
import { generatePkce, randomBase64Url } from '../lib/crypto';
import { exchangeCodeForTokens, parseIdToken } from '../auth/entra';
import { signSession } from '../auth/session';
import { newId } from '../lib/ulid';
import { getDb } from '../db/client';
import { tenants, users, sessions } from '../db/schema';
import { writeAudit } from '../db/audit';
import { Errors } from '../lib/errors';

export const auth = new Hono<AppCtx>();

const STATE_TTL_SECONDS = 600; // 10 min

auth.get('/login', async (c) => {
  const next = c.req.query('next') ?? '/';
  const { verifier, challenge } = await generatePkce();
  const state = randomBase64Url(16);
  await c.env.KV.put(`oauth_state:${state}`, JSON.stringify({ verifier, next }), {
    expirationTtl: STATE_TTL_SECONDS,
  });
  const url = new URL(c.env.ENTRA_AUTHORITY.replace(/\/$/, '') + '/oauth2/v2.0/authorize');
  url.searchParams.set('client_id', c.env.ENTRA_CLIENT_ID);
  url.searchParams.set('response_type', 'code');
  url.searchParams.set('redirect_uri', c.env.API_BASE_URL + '/auth/callback');
  url.searchParams.set('scope', 'openid profile email offline_access');
  url.searchParams.set('response_mode', 'query');
  url.searchParams.set('state', state);
  url.searchParams.set('code_challenge', challenge);
  url.searchParams.set('code_challenge_method', 'S256');
  return c.redirect(url.toString(), 302);
});

auth.get('/callback', async (c) => {
  const code = c.req.query('code');
  const state = c.req.query('state');
  if (!code || !state) throw Errors.BadRequest('missing code or state');
  const stateRaw = await c.env.KV.get(`oauth_state:${state}`);
  if (!stateRaw) throw Errors.BadRequest('invalid or expired state');
  await c.env.KV.delete(`oauth_state:${state}`);
  const { verifier, next } = JSON.parse(stateRaw) as { verifier: string; next: string };

  const tokens = await exchangeCodeForTokens({
    authority: c.env.ENTRA_AUTHORITY,
    clientId: c.env.ENTRA_CLIENT_ID,
    clientSecret: c.env.ENTRA_CLIENT_SECRET,
    code,
    redirectUri: c.env.API_BASE_URL + '/auth/callback',
    codeVerifier: verifier,
  });
  const claims = parseIdToken(tokens.id_token);

  const db = getDb(c.env.DB);

  // 1. Find or refuse tenant by Entra tid
  const tenantRows = await db.select().from(tenants).where(eq(tenants.entraTenantId, claims.tid)).limit(1);
  const tenant = tenantRows[0];
  if (!tenant) {
    throw Errors.Forbidden(`no OrderFlow tenant linked to Entra tenant ${claims.tid}`);
  }

  // 2. Upsert user
  const existing = await db
    .select()
    .from(users)
    .where(and(eq(users.tenantId, tenant.id), eq(users.entraOid, claims.oid)))
    .limit(1);
  let userRow = existing[0];
  if (!userRow) {
    const id = newId();
    await db.insert(users).values({
      id,
      tenantId: tenant.id,
      email: claims.email,
      displayName: claims.name,
      entraOid: claims.oid,
      role: 'viewer',
      status: 'active',
      createdAt: new Date(),
    });
    userRow = (await db.select().from(users).where(eq(users.id, id)).limit(1))[0]!;
  }

  // 3. Create session row
  const sessionId = newId();
  const sessionTtlSec = 60 * 60 * 24 * 7;
  await db.insert(sessions).values({
    id: sessionId,
    userId: userRow.id,
    tenantId: tenant.id,
    deviceLabel: c.req.header('user-agent') ?? null,
    expiresAt: new Date(Date.now() + sessionTtlSec * 1000),
    lastActiveAt: new Date(),
    ip: c.req.header('cf-connecting-ip') ?? null,
    userAgent: c.req.header('user-agent') ?? null,
  });

  // 4. Sign session JWT
  const jwt = await signSession(
    { sub: userRow.id, tid: tenant.id, role: userRow.role, sid: sessionId },
    c.env.SESSION_JWT_SECRET,
    sessionTtlSec,
  );

  // 5. Audit
  await writeAudit(db, {
    tenantId: tenant.id,
    actorUserId: userRow.id,
    via: 'web',
    action: 'auth.login',
    ip: c.req.header('cf-connecting-ip') ?? undefined,
    userAgent: c.req.header('user-agent') ?? undefined,
  });

  // 6. Set cookie + redirect to web app
  const cookie = [
    `${c.env.SESSION_COOKIE_NAME}=${jwt}`,
    'Path=/',
    'HttpOnly',
    'SameSite=Lax',
    `Max-Age=${sessionTtlSec}`,
    c.env.ENVIRONMENT === 'dev' ? '' : 'Secure',
  ]
    .filter(Boolean)
    .join('; ');

  return new Response(null, {
    status: 302,
    headers: {
      location: c.env.APP_BASE_URL + (next.startsWith('/') ? next : '/'),
      'set-cookie': cookie,
    },
  });
});

auth.post('/logout', async (c) => {
  return new Response(null, {
    status: 204,
    headers: {
      'set-cookie': `${c.env.SESSION_COOKIE_NAME}=; Path=/; HttpOnly; Max-Age=0; SameSite=Lax`,
    },
  });
});
```

- [ ] **Step 4: Add `/api/me` and mount auth routes — modify `apps/api/src/index.ts`**

```ts
import { Hono } from 'hono';
import type { AppCtx } from './env';
import { health } from './routes/health';
import { auth } from './routes/auth';
import { requireAuth } from './auth/middleware';
import { getDb } from './db/client';
import { users } from './db/schema';
import { eq, and } from 'drizzle-orm';
import { AppError } from './lib/errors';

const app = new Hono<AppCtx>();

app.onError((err, c) => {
  if (err instanceof AppError) {
    return c.json({ error: err.code, message: err.message, details: err.details }, err.status);
  }
  console.error('unhandled', err);
  return c.json({ error: 'internal_error', message: 'Internal server error' }, 500);
});

app.route('/api/health', health);
app.route('/auth', auth);

app.use('/api/*', requireAuth);

app.get('/api/me', async (c) => {
  const db = getDb(c.env.DB);
  const rows = await db
    .select()
    .from(users)
    .where(and(eq(users.id, c.var.userId!), eq(users.tenantId, c.var.tenantId!)))
    .limit(1);
  const u = rows[0];
  if (!u) return c.json({ error: 'not_found' }, 404);
  return c.json({
    id: u.id,
    email: u.email,
    displayName: u.displayName,
    role: u.role,
    tenantId: u.tenantId,
  });
});

export default app;
```

- [ ] **Step 5: Re-run tests**

```bash
bun --cwd apps/api run test -- routes.test.ts
```

Expected: 2/2 PASS.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/routes/auth.ts apps/api/src/index.ts apps/api/tests/auth/routes.test.ts
git commit -m "feat(api): /auth/login + /auth/callback + /auth/logout + /api/me"
```

---

## Task 12: Dev Seed Script + Manual SSO Smoke Test

**Files:**
- Create: `apps/api/src/seed.ts`
- Create: `apps/api/.dev.vars.example`

- [ ] **Step 1: Create `apps/api/src/seed.ts`**

```ts
/**
 * Seeds a single tenant + admin user for local dev.
 * Run with: bun --cwd apps/api wrangler d1 execute orderflow_dev --local --file=seed.sql
 * (or: bun run scripts/seed.ts via a custom wrangler invocation)
 *
 * For convenience we just emit SQL statements that the developer can paste.
 */
import { newId } from './lib/ulid';

const tenantId = 'tenant_dev_01';
const userId = newId();

const sql = [
  `INSERT INTO tenants (id, name, slug, plan, entra_tenant_id, high_value_threshold, locale_default, created_at)
   VALUES ('${tenantId}', 'Dev Tenant', 'dev', 'free', 'REPLACE_WITH_YOUR_ENTRA_TID', 1000000, 'zh-TW', ${Date.now()});`,
  `INSERT INTO users (id, tenant_id, email, display_name, entra_oid, role, status, created_at)
   VALUES ('${userId}', '${tenantId}', 'REPLACE_WITH_YOUR_EMAIL', 'Dev Admin', 'REPLACE_WITH_YOUR_ENTRA_OID', 'admin', 'active', ${Date.now()});`,
];

console.log(sql.join('\n'));
```

- [ ] **Step 2: Create `apps/api/.dev.vars.example`**

```bash
# Copy to .dev.vars (which is gitignored) and fill in real values from Azure portal.

ENTRA_CLIENT_ID="00000000-0000-0000-0000-000000000000"
ENTRA_CLIENT_SECRET="paste-azure-app-secret"
SESSION_JWT_SECRET="generate-with: openssl rand -base64 48"
```

- [ ] **Step 3: Document the manual one-time setup**

Add to `apps/api/.dev.vars.example` at the top:

```
# One-time setup before running `bun run dev`:
#
# 1. In Azure portal:
#    - Create an App registration (multi-tenant or single tenant — your choice)
#    - Add redirect URI: http://localhost:8787/auth/callback
#    - Note the Application (client) ID and Directory (tenant) ID
#    - Generate a client secret
#
# 2. Generate a session secret:
#      openssl rand -base64 48
#
# 3. Copy this file to .dev.vars and fill in the values above.
#
# 4. Generate seed SQL:
#      bun --cwd apps/api run -e ./src/seed.ts > seed.sql
#    Edit seed.sql, replacing REPLACE_WITH_* with:
#      - Your Entra tenant ID (Directory ID)
#      - Your work email
#      - Your Entra Object ID (find in Azure portal → Users → your account → Object ID)
#
# 5. Apply seed:
#      bun --cwd apps/api wrangler d1 execute orderflow_dev --local --file=seed.sql
#
# 6. Start servers (two terminals):
#      bun run dev:api
#      bun run dev:web
#
# 7. Open http://localhost:5173/login → click "Sign in with Microsoft"
```

- [ ] **Step 4: Manual smoke test**

(This step is documentation for the implementer — automated E2E that talks to real M365 is not feasible.)

Document expected behavior in the README to be created in Task 15.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/seed.ts apps/api/.dev.vars.example
git commit -m "chore(api): dev seed script + .dev.vars example with setup docs"
```

---

## Task 13: i18n Setup (zh-TW + en)

**Files:**
- Create: `apps/web/src/i18n/index.ts`
- Create: `apps/web/src/i18n/zh-TW.json`
- Create: `apps/web/src/i18n/en.json`
- Modify: `apps/web/src/main.tsx` (init i18n before render)

- [ ] **Step 1: Create `apps/web/src/i18n/zh-TW.json`**

```json
{
  "app": { "title": "OrderFlow", "loading": "載入中…" },
  "auth": {
    "signIn": "以 Microsoft 帳號登入",
    "signInTitle": "登入 OrderFlow",
    "signInSubtitle": "使用您的公司 Microsoft 365 帳號登入",
    "signOut": "登出",
    "signedOut": "已登出"
  },
  "nav": {
    "dashboard": "儀表板",
    "orders": "訂單",
    "customers": "客戶",
    "reports": "報表",
    "settings": "設定"
  },
  "shell": {
    "tenant": "租戶",
    "role": "角色",
    "roles": {
      "admin": "管理員",
      "sales_manager": "業務主管",
      "sales": "業務",
      "ops": "生產 / 倉管",
      "accountant": "會計",
      "viewer": "檢視者"
    }
  }
}
```

- [ ] **Step 2: Create `apps/web/src/i18n/en.json`**

```json
{
  "app": { "title": "OrderFlow", "loading": "Loading…" },
  "auth": {
    "signIn": "Sign in with Microsoft",
    "signInTitle": "Sign in to OrderFlow",
    "signInSubtitle": "Use your company Microsoft 365 account",
    "signOut": "Sign out",
    "signedOut": "Signed out"
  },
  "nav": {
    "dashboard": "Dashboard",
    "orders": "Orders",
    "customers": "Customers",
    "reports": "Reports",
    "settings": "Settings"
  },
  "shell": {
    "tenant": "Tenant",
    "role": "Role",
    "roles": {
      "admin": "Admin",
      "sales_manager": "Sales Manager",
      "sales": "Sales",
      "ops": "Operations",
      "accountant": "Accountant",
      "viewer": "Viewer"
    }
  }
}
```

- [ ] **Step 3: Create `apps/web/src/i18n/index.ts`**

```ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import zhTW from './zh-TW.json';
import en from './en.json';

void i18n.use(initReactI18next).init({
  resources: { 'zh-TW': { translation: zhTW }, en: { translation: en } },
  lng: 'zh-TW',
  fallbackLng: 'en',
  interpolation: { escapeValue: false },
});

export default i18n;
```

- [ ] **Step 4: Update `apps/web/src/main.tsx` to init i18n**

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App';
import './styles/globals.css';
import './i18n';

const qc = new QueryClient();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={qc}>
      <App />
    </QueryClientProvider>
  </React.StrictMode>,
);
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/i18n apps/web/src/main.tsx
git commit -m "feat(web): i18next setup with zh-TW (default) + en"
```

---

## Task 14: UI Shell + Login Page + Post-Login Dashboard

**Files:**
- Create: `apps/web/src/lib/api.ts`
- Create: `apps/web/src/lib/auth.ts`
- Create: `apps/web/src/components/shell/AppShell.tsx`
- Create: `apps/web/src/components/shell/Sidebar.tsx`
- Create: `apps/web/src/components/shell/TopBar.tsx`
- Create: `apps/web/src/routes/login.tsx`
- Create: `apps/web/src/routes/index.tsx`
- Modify: `apps/web/src/App.tsx`

- [ ] **Step 1: Create `apps/web/src/lib/api.ts`**

```ts
export class ApiError extends Error {
  constructor(public status: number, public code: string, message: string) {
    super(message);
  }
}

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(path, { credentials: 'include', ...init });
  if (!res.ok) {
    let body: { error?: string; message?: string } = {};
    try { body = await res.json(); } catch {}
    throw new ApiError(res.status, body.error ?? 'unknown', body.message ?? res.statusText);
  }
  return (await res.json()) as T;
}
```

- [ ] **Step 2: Create `apps/web/src/lib/auth.ts`**

```ts
import { useQuery } from '@tanstack/react-query';
import { api, ApiError } from './api';

export interface Me {
  id: string;
  email: string;
  displayName: string;
  role: 'admin' | 'sales_manager' | 'sales' | 'ops' | 'accountant' | 'viewer';
  tenantId: string;
}

export function useMe() {
  return useQuery({
    queryKey: ['me'],
    queryFn: async () => {
      try {
        return await api<Me>('/api/me');
      } catch (e) {
        if (e instanceof ApiError && e.status === 401) return null;
        throw e;
      }
    },
    staleTime: 60_000,
  });
}

export function loginUrl(next: string = window.location.pathname) {
  return `/auth/login?next=${encodeURIComponent(next)}`;
}

export async function logout() {
  await fetch('/auth/logout', { method: 'POST', credentials: 'include' });
  window.location.href = '/login';
}
```

- [ ] **Step 3: Create `apps/web/src/components/shell/Sidebar.tsx`**

```tsx
import { useTranslation } from 'react-i18next';

const NAV = [
  { key: 'dashboard', href: '/', enabled: true },
  { key: 'orders', href: '/orders', enabled: false },
  { key: 'customers', href: '/customers', enabled: false },
  { key: 'reports', href: '/reports', enabled: false },
  { key: 'settings', href: '/settings', enabled: false },
] as const;

export function Sidebar() {
  const { t } = useTranslation();
  return (
    <aside className="w-56 shrink-0 border-r border-slate-200 bg-white p-4">
      <div className="mb-6 text-xl font-bold">{t('app.title')}</div>
      <nav className="flex flex-col gap-1">
        {NAV.map((item) => (
          <a
            key={item.key}
            href={item.href}
            aria-disabled={!item.enabled}
            className={`rounded px-3 py-2 text-sm ${
              item.enabled
                ? 'text-slate-700 hover:bg-slate-100'
                : 'cursor-not-allowed text-slate-400'
            }`}
          >
            {t(`nav.${item.key}`)}
          </a>
        ))}
      </nav>
    </aside>
  );
}
```

- [ ] **Step 4: Create `apps/web/src/components/shell/TopBar.tsx`**

```tsx
import { useTranslation } from 'react-i18next';
import { logout, type Me } from '@/lib/auth';

export function TopBar({ me }: { me: Me }) {
  const { t } = useTranslation();
  return (
    <header className="flex items-center justify-between border-b border-slate-200 bg-white px-6 py-3">
      <div className="text-sm text-slate-500">
        {t('shell.tenant')}: <span className="font-medium text-slate-700">{me.tenantId}</span>
        <span className="mx-3">·</span>
        {t('shell.role')}: <span className="font-medium text-slate-700">{t(`shell.roles.${me.role}`)}</span>
      </div>
      <div className="flex items-center gap-3">
        <span className="text-sm text-slate-600">{me.displayName}</span>
        <button
          onClick={() => void logout()}
          className="rounded border border-slate-300 px-3 py-1 text-sm hover:bg-slate-50"
        >
          {t('auth.signOut')}
        </button>
      </div>
    </header>
  );
}
```

- [ ] **Step 5: Create `apps/web/src/components/shell/AppShell.tsx`**

```tsx
import type { ReactNode } from 'react';
import { Sidebar } from './Sidebar';
import { TopBar } from './TopBar';
import type { Me } from '@/lib/auth';

export function AppShell({ me, children }: { me: Me; children: ReactNode }) {
  return (
    <div className="flex h-full">
      <Sidebar />
      <div className="flex flex-1 flex-col">
        <TopBar me={me} />
        <main className="flex-1 overflow-auto p-6">{children}</main>
      </div>
    </div>
  );
}
```

- [ ] **Step 6: Create `apps/web/src/routes/login.tsx`**

```tsx
import { useTranslation } from 'react-i18next';
import { loginUrl } from '@/lib/auth';

export function LoginPage() {
  const { t } = useTranslation();
  const next = new URLSearchParams(window.location.search).get('next') ?? '/';
  return (
    <div className="flex h-full items-center justify-center bg-slate-50">
      <div className="w-full max-w-sm rounded-lg border border-slate-200 bg-white p-8 shadow-sm">
        <h1 className="mb-2 text-2xl font-bold">{t('auth.signInTitle')}</h1>
        <p className="mb-6 text-sm text-slate-600">{t('auth.signInSubtitle')}</p>
        <a
          href={loginUrl(next)}
          className="block w-full rounded bg-slate-900 px-4 py-2 text-center text-sm font-medium text-white hover:bg-slate-800"
        >
          {t('auth.signIn')}
        </a>
      </div>
    </div>
  );
}
```

- [ ] **Step 7: Create `apps/web/src/routes/index.tsx`**

```tsx
import { useTranslation } from 'react-i18next';
import { AppShell } from '@/components/shell/AppShell';
import type { Me } from '@/lib/auth';

export function DashboardPage({ me }: { me: Me }) {
  const { t } = useTranslation();
  return (
    <AppShell me={me}>
      <h1 className="mb-2 text-2xl font-bold">{t('nav.dashboard')}</h1>
      <p className="text-slate-600">
        Welcome, <strong>{me.displayName}</strong>. Phase 0 shell is live. Phase 1 features begin
        rolling in next sprint.
      </p>
    </AppShell>
  );
}
```

- [ ] **Step 8: Replace `apps/web/src/App.tsx` with router-less guard**

```tsx
import { useEffect } from 'react';
import { useMe } from '@/lib/auth';
import { LoginPage } from '@/routes/login';
import { DashboardPage } from '@/routes/index';

export default function App() {
  const { data: me, isLoading } = useMe();
  const path = window.location.pathname;

  useEffect(() => {
    if (!isLoading && !me && path !== '/login') {
      window.location.href = `/login?next=${encodeURIComponent(path)}`;
    }
  }, [isLoading, me, path]);

  if (isLoading) return <div className="p-8">Loading…</div>;
  if (!me) return <LoginPage />;
  return <DashboardPage me={me} />;
}
```

> **Phase 1 will swap this for TanStack Router with file-based routing.** For Phase 0 we use a minimal switch to avoid pulling routing complexity into the foundation.

- [ ] **Step 9: Run dev servers and visit manually**

```bash
# Terminal 1
bun run dev:api
# Terminal 2
bun run dev:web
```

Open `http://localhost:5173/`. Without a session you should be redirected to `/login`. The sign-in button targets `/auth/login` (proxied to the Worker on 8787).

Stop servers with Ctrl+C.

- [ ] **Step 10: Commit**

```bash
git add apps/web/src
git commit -m "feat(web): shell + login + dashboard with i18n + /api/me integration"
```

---

## Task 15: Playwright E2E (Login Redirect + Health) + README

**Files:**
- Create: `apps/web/playwright.config.ts`
- Create: `apps/web/tests/e2e/auth.spec.ts`
- Create: `orderflow/README.md`

- [ ] **Step 1: Create `apps/web/playwright.config.ts`**

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30_000,
  fullyParallel: false,
  retries: 0,
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
});
```

- [ ] **Step 2: Create `apps/web/tests/e2e/auth.spec.ts`**

```ts
import { test, expect } from '@playwright/test';

test('unauthenticated home redirects to /login', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveURL(/\/login/);
  await expect(page.getByRole('heading', { name: /登入 OrderFlow|Sign in to OrderFlow/ })).toBeVisible();
});

test('login page sign-in link points to /auth/login', async ({ page }) => {
  await page.goto('/login');
  const link = page.getByRole('link', { name: /Microsoft/ });
  const href = await link.getAttribute('href');
  expect(href).toMatch(/^\/auth\/login(\?next=.*)?$/);
});

test('api health endpoint responds when API is up', async ({ request }) => {
  const res = await request.get('http://localhost:8787/api/health');
  expect(res.ok()).toBeTruthy();
  const body = await res.json();
  expect(body.ok).toBe(true);
});
```

- [ ] **Step 3: Run E2E (assuming dev servers are running)**

```bash
# Terminal 1
bun run dev:api
# Terminal 2
bun run dev:web
# Terminal 3
bun --cwd apps/web exec playwright install chromium
bun --cwd apps/web run test:e2e
```

Expected: 3/3 PASS.

- [ ] **Step 4: Create `orderflow/README.md`**

```markdown
# OrderFlow

Multi-tenant business project & shipment tracking system. Phase 0 (Foundation) — local development only.

## Prerequisites

- Bun ≥ 1.1
- A Microsoft Entra app registration with redirect URI `http://localhost:8787/auth/callback`

## Setup

```bash
bun install

# 1. Create local D1
bun --cwd apps/api wrangler d1 create orderflow_dev
# (paste the printed database_id into apps/api/wrangler.toml)

# 2. Apply migrations
bun --cwd apps/api run db:migrate

# 3. Configure secrets
cp apps/api/.dev.vars.example apps/api/.dev.vars
# Fill in ENTRA_CLIENT_ID, ENTRA_CLIENT_SECRET, SESSION_JWT_SECRET

# 4. Seed your dev tenant + admin user
bun --cwd apps/api bun run src/seed.ts > seed.sql
# Edit seed.sql, replacing REPLACE_WITH_* placeholders. Then:
bun --cwd apps/api wrangler d1 execute orderflow_dev --local --file=seed.sql
```

## Run

```bash
# Terminal 1
bun run dev:api      # Worker on http://localhost:8787

# Terminal 2
bun run dev:web      # Vite on http://localhost:5173
```

Open `http://localhost:5173`. You'll be redirected to `/login` and can sign in with Microsoft.

## Tests

```bash
bun run test                     # api + web unit tests
bun --cwd apps/web run test:e2e  # Playwright (requires both servers running)
bun run typecheck
```

## Project layout

See `docs/superpowers/specs/2026-04-30-orderflow-design.md` §13.

## Known Phase-0 limitations (resolved in Phase 1+)

- ID token signature is **not** verified against Entra JWKS (we trust the TLS-secured token endpoint response). This is acceptable for dev; Phase 1 hardening adds JWKS verification.
- No business features (orders, customers, stages) — only the auth/shell foundation.
- TanStack Router file-based routing is deferred — App.tsx uses a minimal switch.
```

- [ ] **Step 5: Commit**

```bash
git add apps/web/playwright.config.ts apps/web/tests/e2e README.md
git commit -m "test(web): Playwright E2E for login redirect + health + README"
```

---

## Task 16: Final Phase-0 Verification

- [ ] **Step 1: Run all unit tests**

```bash
cd /Users/swryociao/orderflow
bun run test
```

Expected: all api tests pass.

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: no errors.

- [ ] **Step 3: Run Playwright (with dev servers up)**

```bash
# Terminal 1: bun run dev:api
# Terminal 2: bun run dev:web
bun --cwd apps/web run test:e2e
```

Expected: 3/3 pass.

- [ ] **Step 4: Manual SSO smoke test**

1. Sign in via `http://localhost:5173/login` → redirected to Microsoft → back to dashboard
2. Dashboard shows your `displayName`, tenant id, role badge
3. `wrangler d1 execute orderflow_dev --local --command "SELECT action, actor_user_id, created_at FROM audit_log ORDER BY created_at DESC LIMIT 5;"` shows an `auth.login` row
4. Click "Sign out" → redirected to `/login` → trying `/` again redirects to login

Document the screenshots in `docs/phase-0-verification.md` (optional).

- [ ] **Step 5: Tag the milestone**

```bash
git tag -a phase-0-complete -m "Phase 0 foundation complete"
git log --oneline -20
```

- [ ] **Step 6: Final commit (if README updates)**

```bash
git add -A
git diff --cached --quiet || git commit -m "chore: phase 0 verification doc"
```

---

## Done Criteria

Phase 0 is **complete** when:

1. `bun run test` is green (api unit tests across crypto, session, entra, audit, middleware, rbac, routes).
2. `bun run typecheck` is green for both apps.
3. Playwright E2E tests pass.
4. A real M365 user can sign in, sees the shell with their tenant/role, and the audit log records the login.
5. Sign-out clears the cookie and re-prompts login.

## Out of Scope for Phase 0 (deferred)

- ID token JWKS verification (Phase 1 W1 hardening)
- Sessions revocation list (Phase 2 with device cap)
- Self-service tenant signup (Phase 3)
- Order / customer / stage / document / alert features (Phase 1 weeks 1-6)
- TanStack Router file-based routing (Phase 1 W1)
- shadcn/ui primitives setup (Phase 1 W1, when forms appear)

## Risks & Notes

- **Entra app registration friction:** budget 30-60 min for first-time Azure portal setup; redirect URI mismatch is the most common error.
- **D1 migrations on multi-developer team:** if/when a second developer joins, document `wrangler d1 migrations apply` in onboarding (already in README).
- **Cookie SameSite=Lax** works for localhost cross-port (5173 ↔ 8787) because the Vite proxy forwards `/auth` to the Worker, keeping the session cookie same-origin from the browser's POV.
