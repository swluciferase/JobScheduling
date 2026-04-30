# OrderFlow Phase 0: Foundation Implementation Plan (v0.2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the QMS-grade foundation for OrderFlow — single-company monorepo, M365 SSO behind Cloudflare Access Zero Trust gate, hash-chained immutable audit log, electronic signature framework (21 CFR Part 11), 7-role RBAC with separation-of-duties helpers, and the UI shell — so Phase 1 can start adding business features without redoing infrastructure or compliance scaffolding.

**Architecture:** Bun workspace monorepo with `apps/web` (React + Vite + Tailwind + shadcn) and `apps/api` (Cloudflare Workers + Hono + Drizzle ORM + D1). **Single-tenant**: no `tenant_id` columns; configuration values (`org_name`, `entra_tenant_id`, etc.) live in a single-row `organization` table. Authentication is two-layer: (1) Cloudflare Access verifies M365 + MFA before any traffic reaches the Worker, (2) application reads `Cf-Access-Jwt-Assertion` header to identify user, then issues a short-lived application session JWT (HttpOnly, 2h absolute, 30min idle). Audit log uses sha256 hash-chain with strict monotonic `seq`. E-signatures require re-authentication and write a paired audit row.

**Tech Stack:** Bun, TypeScript, React 18, Vite, Tailwind, shadcn/ui, TanStack Query, TanStack Router, Hono, Drizzle ORM, Cloudflare D1 (Wrangler local), `jose` (JWT + JWKS), Vitest, Playwright, `@cloudflare/cloudflare-access-jwt` (or manual JWKS verify).

**Phase boundary:** Ends when (a) a real M365 user passes Cloudflare Access + completes Entra OIDC PKCE → enters app shell with their name + role visible; (b) `audit_log` records the login as seq=N with verifiable hash chain; (c) the Phase 0 demo "签个名" button triggers the e-signature flow (re-auth + intent confirmation), writes `e_signatures` + `audit_log` rows, and renders a signature manifest snippet. **No business features (orders/customers/NCR) implemented in Phase 0** — those start in Phase 1.

---

## Reference Spec

Always cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §2 法規對照表 (Phase 0 maps to: 11.10(d)/11.10(e)/11.10(g)/11.50/11.70/11.100/11.200)
- §3 角色與職責分離 (7 roles + SoD framework, full matrix wired in Phase 1)
- §8 認證、SSO、存取控制 (Cloudflare Access + M365 + session policy)
- §9 電子簽章 (framework only; concrete uses in Phase 1+)
- §10 稽核軌跡 (hash chain + immutability triggers)
- §19 目錄結構

---

## File Structure

```
orderflow/
├── package.json                        # workspace root
├── bun.lockb
├── tsconfig.base.json
├── .gitignore
├── .editorconfig
├── docs/                                # already exists
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
│   │   │   │   ├── shell/
│   │   │   │   │   ├── AppShell.tsx
│   │   │   │   │   ├── Sidebar.tsx
│   │   │   │   │   └── TopBar.tsx
│   │   │   │   └── esig/
│   │   │   │       ├── EsigDialog.tsx
│   │   │   │       └── EsigDemoButton.tsx
│   │   │   ├── lib/
│   │   │   │   ├── api.ts
│   │   │   │   ├── auth.ts
│   │   │   │   └── esig.ts
│   │   │   ├── i18n/
│   │   │   │   ├── index.ts
│   │   │   │   ├── zh-TW.json
│   │   │   │   └── en.json
│   │   │   └── styles/globals.css
│   │   └── tests/
│   │       └── e2e/
│   │           ├── auth.spec.ts
│   │           ├── esig-demo.spec.ts
│   │           └── audit-chain.spec.ts
│   └── api/
│       ├── package.json
│       ├── tsconfig.json
│       ├── wrangler.toml
│       ├── drizzle.config.ts
│       ├── src/
│       │   ├── index.ts                 # Hono app entry
│       │   ├── env.ts
│       │   ├── routes/
│       │   │   ├── health.ts
│       │   │   ├── auth.ts
│       │   │   └── esig.ts              # POST /esig/sign + GET /esig/:id
│       │   ├── auth/
│       │   │   ├── cf-access.ts         # verify Cf-Access-Jwt-Assertion
│       │   │   ├── entra.ts             # OIDC discovery + token exchange + JWKS verify
│       │   │   ├── session.ts           # app session JWT
│       │   │   ├── reauth.ts            # re-auth for e-sig
│       │   │   ├── middleware.ts        # auth context injection
│       │   │   └── rbac.ts              # 7-role matrix + SoD helpers
│       │   ├── db/
│       │   │   ├── schema/
│       │   │   │   ├── organization.ts
│       │   │   │   ├── users.ts
│       │   │   │   ├── teams.ts
│       │   │   │   ├── sessions.ts
│       │   │   │   ├── audit_log.ts
│       │   │   │   ├── e_signatures.ts
│       │   │   │   ├── access_reviews.ts
│       │   │   │   ├── change_records.ts
│       │   │   │   └── index.ts
│       │   │   ├── migrations/
│       │   │   │   ├── 0000_init.sql
│       │   │   │   └── 0001_immutable_triggers.sql
│       │   │   ├── client.ts            # getDb()
│       │   │   └── audit.ts             # hash-chain writer
│       │   ├── crypto/
│       │   │   ├── pkce.ts
│       │   │   ├── hmac.ts
│       │   │   ├── sha256.ts
│       │   │   ├── canonical-json.ts    # deterministic JSON for hashing
│       │   │   └── hash-chain.ts        # row_hash compute + verify chain
│       │   ├── lib/
│       │   │   ├── ulid.ts
│       │   │   └── errors.ts
│       │   └── seed.ts                  # dev seed: org + demo users (7 roles)
│       └── tests/
│           ├── auth/
│           ├── crypto/
│           │   ├── canonical-json.test.ts
│           │   └── hash-chain.test.ts
│           ├── db/
│           │   ├── audit-chain.test.ts
│           │   ├── immutability.test.ts
│           │   └── e-signatures.test.ts
│           └── routes/
│               ├── auth.test.ts
│               └── esig.test.ts
└── packages/
    └── shared/
        ├── package.json
        └── src/
            ├── roles.ts                 # role enum + SoD rules
            ├── esig-meanings.ts
            └── index.ts
```

---

## Task List Overview

Week 1: Tasks 1-9 (skeleton, web/api scaffold, schema, immutability, crypto, hash-chain audit, sessions)
Week 2: Tasks 10-18 (Cloudflare Access verify, Entra OIDC, app session, RBAC, SoD, e-sig flow, UI shell, demo, E2E, verification)

---

### Task 1: Monorepo Skeleton

**Files:**
- Create: `package.json`, `tsconfig.base.json`, `.gitignore`, `.editorconfig`

- [ ] **Step 1: Verify project root state**
```bash
cd /Users/swryociao/orderflow && git status && ls -la
```
Expected: working tree clean, only `docs/` exists.

- [ ] **Step 2: Workspace `package.json`**
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
  "engines": { "bun": ">=1.1.0" }
}
```

- [ ] **Step 3: `tsconfig.base.json`** with `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `target: ES2022`, `moduleResolution: bundler`.

- [ ] **Step 4: `.gitignore`** (`node_modules`, `dist`, `.wrangler`, `.dev.vars`, `*.log`, `.env*`).

- [ ] **Step 5: `.editorconfig`** (LF, UTF-8, 2-space).

- [ ] **Step 6: Commit**
```bash
git add package.json tsconfig.base.json .gitignore .editorconfig
git commit -m "feat(repo): bun workspace skeleton"
```

---

### Task 2: Shared package (roles + e-sig meanings)

**Files:**
- Create: `packages/shared/package.json`, `tsconfig.json`
- Create: `packages/shared/src/roles.ts`, `esig-meanings.ts`, `index.ts`
- Test: `packages/shared/test/roles.test.ts`

- [ ] **Step 1: Failing test**
```ts
import { ROLES, ROLE_RANK, isHigherRole, ESIG_MEANINGS } from '@orderflow/shared';
it('exports 7 roles in canonical order', () => {
  expect(ROLES).toEqual(['admin','qa_manager','sales_manager','sales','ops','accountant','viewer']);
});
it('ROLE_RANK assigns numeric weight', () => {
  expect(ROLE_RANK.admin).toBeGreaterThan(ROLE_RANK.viewer);
});
it('isHigherRole(qa_manager, sales) === true', () => {
  expect(isHigherRole('qa_manager','sales')).toBe(true);
});
it('ESIG_MEANINGS = review|approval|responsibility|authorship', () => {
  expect(ESIG_MEANINGS).toEqual(['review','approval','responsibility','authorship']);
});
```

- [ ] **Step 2: Implement**
```ts
export const ROLES = ['admin','qa_manager','sales_manager','sales','ops','accountant','viewer'] as const;
export type Role = typeof ROLES[number];
export const ROLE_RANK: Record<Role, number> = { admin:100, qa_manager:90, sales_manager:80, sales:50, ops:50, accountant:50, viewer:10 };
export const isHigherRole = (a: Role, b: Role) => ROLE_RANK[a] > ROLE_RANK[b];
export const ESIG_MEANINGS = ['review','approval','responsibility','authorship'] as const;
export type EsigMeaning = typeof ESIG_MEANINGS[number];
```

- [ ] **Step 3: Run test → PASS**

- [ ] **Step 4: Commit**

---

### Task 3: Web app scaffold

**Files:**
- Create: `apps/web/package.json`, `tsconfig.json`, `vite.config.ts`, `tailwind.config.ts`, `postcss.config.js`, `index.html`, `src/main.tsx`, `src/App.tsx`, `src/styles/globals.css`

- [ ] **Step 1: bun create vite or manual**

`apps/web/package.json`:
```json
{
  "name": "@orderflow/web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --port 5173",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest --run",
    "typecheck": "tsc --noEmit",
    "e2e": "playwright test"
  },
  "dependencies": {
    "react": "^18.3.0", "react-dom": "^18.3.0",
    "@tanstack/react-query": "^5", "@tanstack/react-router": "^1",
    "react-hook-form": "^7", "zod": "^3", "@hookform/resolvers": "^3",
    "clsx": "^2", "tailwind-merge": "^2", "lucide-react": "^0",
    "@orderflow/shared": "workspace:*"
  },
  "devDependencies": {
    "vite": "^5", "@vitejs/plugin-react": "^4",
    "tailwindcss": "^3", "postcss": "^8", "autoprefixer": "^10",
    "typescript": "^5", "vitest": "^1", "@testing-library/react": "^16",
    "@playwright/test": "^1"
  }
}
```

- [ ] **Step 2: Vite + Tailwind config** (set `host: 'localhost'`, react plugin, Tailwind preset).

- [ ] **Step 3: Smoke run**
```bash
cd apps/web && bun install && bun run dev
```
Expected: `http://localhost:5173` shows "OrderFlow".

- [ ] **Step 4: Commit**

---

### Task 4: API app scaffold (Hono + Drizzle + Wrangler local D1)

**Files:**
- Create: `apps/api/package.json`, `tsconfig.json`, `wrangler.toml`, `drizzle.config.ts`, `src/index.ts`, `src/env.ts`, `src/routes/health.ts`

- [ ] **Step 1: package.json**
```json
{
  "name": "@orderflow/api",
  "scripts": {
    "dev": "wrangler dev --local --port 8787",
    "deploy": "wrangler deploy",
    "test": "vitest --run",
    "typecheck": "tsc --noEmit",
    "db:generate": "drizzle-kit generate",
    "db:migrate:local": "wrangler d1 migrations apply orderflow --local",
    "seed:dev": "bun src/seed.ts"
  },
  "dependencies": {
    "hono": "^4", "@hono/zod-validator": "^0",
    "drizzle-orm": "^0.30", "drizzle-kit": "^0.20",
    "jose": "^5", "zod": "^3",
    "@orderflow/shared": "workspace:*"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4", "wrangler": "^3",
    "vitest": "^1", "miniflare": "^3", "typescript": "^5"
  }
}
```

- [ ] **Step 2: wrangler.toml**
```toml
name = "orderflow-api"
main = "src/index.ts"
compatibility_date = "2026-04-30"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "orderflow"
database_id = "REPLACE_WITH_REAL_ID"

[vars]
ENV = "local"
SESSION_IDLE_MINUTES = "30"
SESSION_ABSOLUTE_HOURS = "2"
ENTRA_TENANT_ID = "REPLACE"
ENTRA_CLIENT_ID = "REPLACE"
ENTRA_REDIRECT_URI = "http://localhost:5173/auth/callback"
CF_ACCESS_TEAM_DOMAIN = "REPLACE"
CF_ACCESS_AUD = "REPLACE"

# secrets via `wrangler secret put`:
# ENTRA_CLIENT_SECRET, JWT_SIGNING_KEY, AUDIT_HASH_PEPPER
```

- [ ] **Step 3: Hono entry + /health**
```ts
import { Hono } from 'hono';
import health from './routes/health';
const app = new Hono<{ Bindings: Env }>();
app.route('/health', health);
export default app;
```

- [ ] **Step 4: Smoke**
```bash
cd apps/api && bun install && bun run dev
curl http://localhost:8787/health
```
Expected: `{"ok":true,"version":"0.0.1"}`

- [ ] **Step 5: Commit**

---

### Task 5: Database schema + immutability triggers

**Files:**
- Create: `apps/api/src/db/schema/organization.ts`, `users.ts`, `teams.ts`, `sessions.ts`, `audit_log.ts`, `e_signatures.ts`, `access_reviews.ts`, `change_records.ts`, `index.ts`
- Create: `apps/api/src/db/migrations/0000_init.sql`, `0001_immutable_triggers.sql`
- Create: `apps/api/drizzle.config.ts`
- Test: `apps/api/test/db/schema.test.ts`

- [ ] **Step 1: Failing schema test**
```ts
import * as schema from '@/db/schema';
it('organization is single-row table with org_name + entra_tenant_id', () => {
  expect(Object.keys(schema.organization)).toEqual(expect.arrayContaining(['id','orgName','entraTenantId']));
});
it('users has role + disabled + completed_trainings_json', () => {
  expect(Object.keys(schema.users)).toEqual(expect.arrayContaining(['id','email','displayName','role','disabled','completedTrainingsJson']));
});
it('audit_log has seq + prev_hash + row_hash NOT NULL', () => {
  expect(Object.keys(schema.auditLog)).toEqual(expect.arrayContaining(['seq','prevHash','rowHash']));
});
it('e_signatures has user_name_snapshot, record_hash, prev_audit_hash', () => {
  expect(Object.keys(schema.eSignatures)).toEqual(expect.arrayContaining(['userNameSnapshot','recordHash','prevAuditHash']));
});
```

- [ ] **Step 2: SQL migration `0000_init.sql`**
```sql
-- Single-row organization config
CREATE TABLE organization (
  id INTEGER PRIMARY KEY CHECK (id = 1),
  org_name TEXT NOT NULL,
  entra_tenant_id TEXT NOT NULL,
  cf_access_team_domain TEXT,
  cf_access_aud TEXT,
  retention_years INTEGER NOT NULL DEFAULT 7,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE users (
  id TEXT PRIMARY KEY,                    -- ULID
  email TEXT NOT NULL UNIQUE,
  entra_oid TEXT UNIQUE,                  -- Entra objectId
  display_name TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('admin','qa_manager','sales_manager','sales','ops','accountant','viewer')),
  team_id TEXT,
  managed_team_ids_json TEXT NOT NULL DEFAULT '[]',
  notification_prefs_json TEXT NOT NULL DEFAULT '{}',
  completed_trainings_json TEXT NOT NULL DEFAULT '[]',
  disabled INTEGER NOT NULL DEFAULT 0,
  last_login_at TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_disabled ON users(disabled);

CREATE TABLE teams (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  manager_user_id TEXT,
  created_at TEXT NOT NULL
);

CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  issued_at TEXT NOT NULL,
  last_active_at TEXT NOT NULL,
  expires_at TEXT NOT NULL,
  ip_address TEXT,
  user_agent TEXT,
  revoked_at TEXT,
  cf_access_jti TEXT
);
CREATE INDEX idx_sessions_user ON sessions(user_id, expires_at);

CREATE TABLE audit_log (
  id TEXT PRIMARY KEY,                    -- ULID
  seq INTEGER NOT NULL UNIQUE,            -- monotonic
  ts TEXT NOT NULL,                       -- ISO UTC
  actor_user_id TEXT,
  actor_source TEXT NOT NULL CHECK (actor_source IN ('web','api','mcp','system','cron')),
  ip_address TEXT,
  user_agent TEXT,
  action TEXT NOT NULL,
  record_type TEXT,
  record_id TEXT,
  before_json TEXT,
  after_json TEXT,
  metadata_json TEXT,
  prev_hash TEXT NOT NULL,
  row_hash TEXT NOT NULL UNIQUE
);
CREATE INDEX idx_audit_record ON audit_log(record_type, record_id, ts);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id, ts);
CREATE INDEX idx_audit_seq ON audit_log(seq);
CREATE INDEX idx_audit_action ON audit_log(action, ts);

CREATE TABLE e_signatures (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  user_name_snapshot TEXT NOT NULL,
  user_role_snapshot TEXT NOT NULL,
  record_type TEXT NOT NULL,
  record_id TEXT NOT NULL,
  meaning TEXT NOT NULL CHECK (meaning IN ('review','approval','responsibility','authorship')),
  reason TEXT,
  record_hash TEXT NOT NULL,
  signed_at TEXT NOT NULL,
  reauth_method TEXT NOT NULL CHECK (reauth_method IN ('m365_password','m365_mfa_step_up')),
  ip_address TEXT,
  user_agent TEXT,
  prev_audit_hash TEXT,
  audit_id TEXT NOT NULL UNIQUE REFERENCES audit_log(id),
  superseded_by TEXT REFERENCES e_signatures(id)
);
CREATE INDEX idx_esig_record ON e_signatures(record_type, record_id);
CREATE INDEX idx_esig_user ON e_signatures(user_id);

CREATE TABLE access_reviews (
  id TEXT PRIMARY KEY,
  period TEXT NOT NULL,                   -- 'YYYY-Q[1-4]'
  reviewer_id TEXT NOT NULL REFERENCES users(id),
  status TEXT NOT NULL CHECK (status IN ('open','submitted','closed')),
  data_json TEXT NOT NULL,
  esig_id TEXT REFERENCES e_signatures(id),
  created_at TEXT NOT NULL,
  closed_at TEXT
);

CREATE TABLE change_records (
  id TEXT PRIMARY KEY,
  change_no TEXT NOT NULL UNIQUE,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  risk_level TEXT NOT NULL CHECK (risk_level IN ('low','medium','high')),
  affected_area TEXT,
  pr_url TEXT,
  commit_sha TEXT,
  status TEXT NOT NULL,
  esig_admin_id TEXT REFERENCES e_signatures(id),
  esig_qa_id TEXT REFERENCES e_signatures(id),
  created_by TEXT NOT NULL REFERENCES users(id),
  created_at TEXT NOT NULL,
  approved_at TEXT,
  deployed_at TEXT
);
```

- [ ] **Step 3: SQL migration `0001_immutable_triggers.sql`**
```sql
CREATE TRIGGER no_audit_update BEFORE UPDATE ON audit_log
BEGIN SELECT RAISE(ABORT, 'audit_log is immutable'); END;
CREATE TRIGGER no_audit_delete BEFORE DELETE ON audit_log
BEGIN SELECT RAISE(ABORT, 'audit_log is immutable'); END;
CREATE TRIGGER no_esig_update BEFORE UPDATE ON e_signatures
WHEN OLD.superseded_by IS NULL AND NEW.superseded_by IS NULL
BEGIN SELECT RAISE(ABORT, 'e_signatures core fields are immutable'); END;
CREATE TRIGGER no_esig_delete BEFORE DELETE ON e_signatures
BEGIN SELECT RAISE(ABORT, 'e_signatures cannot be deleted'); END;
```

> Note: `e_signatures` allows updating `superseded_by` only (link to newer sig); the `WHEN` clause permits that single transition.

- [ ] **Step 4: Drizzle schemas** mirror SQL.

- [ ] **Step 5: drizzle.config.ts** points to `src/db/schema/index.ts` + dialect=`sqlite`.

- [ ] **Step 6: Apply locally**
```bash
cd apps/api && bunx wrangler d1 create orderflow --local
# Copy database_id into wrangler.toml, then:
bun run db:migrate:local
```

- [ ] **Step 7: Test PASS**

- [ ] **Step 8: Commit**

---

### Task 6: Crypto helpers — canonical JSON + sha256 + hash chain

**Files:**
- Create: `apps/api/src/crypto/canonical-json.ts`, `sha256.ts`, `hash-chain.ts`, `pkce.ts`, `hmac.ts`
- Test: `apps/api/test/crypto/canonical-json.test.ts`, `hash-chain.test.ts`

- [ ] **Step 1: Failing canonical-json test**
```ts
it('canonical JSON sorts keys recursively', () => {
  expect(canonicalJson({ b:2, a:{ y:1, x:2 } })).toBe('{"a":{"x":2,"y":1},"b":2}');
});
it('null preserved, undefined dropped', () => {
  expect(canonicalJson({ a:null, b:undefined as any, c:1 })).toBe('{"a":null,"c":1}');
});
it('arrays preserve order', () => {
  expect(canonicalJson([3,1,2])).toBe('[3,1,2]');
});
```

- [ ] **Step 2: Implement** stable stringifier (recursive, key sort, drop undefined).

- [ ] **Step 3: Failing hash-chain test**
```ts
it('computeRowHash deterministic given identical input', () => {
  const a = computeRowHash({ seq:1, ts:'2026-04-30T00:00:00Z', action:'x', prev_hash:'GENESIS' });
  const b = computeRowHash({ seq:1, ts:'2026-04-30T00:00:00Z', action:'x', prev_hash:'GENESIS' });
  expect(a).toBe(b);
});
it('verifyChain returns ok=true for genuine chain, ok=false + breakAt on tampered row', async () => {
  const rows = buildSampleChain(5);
  expect((await verifyChain(rows)).ok).toBe(true);
  rows[2].after_json = '{"tampered":true}';
  const r = await verifyChain(rows);
  expect(r.ok).toBe(false);
  expect(r.breakAt).toBe(2);
});
it('GENESIS is fixed string "0".repeat(64)', () => {
  expect(GENESIS_PREV_HASH).toBe('0'.repeat(64));
});
```

- [ ] **Step 4: Implement** using WebCrypto `subtle.digest('SHA-256', ...)`. `prev_hash` for seq=1 = GENESIS.

- [ ] **Step 5: PKCE + HMAC helpers** (`generateCodeVerifier`, `codeChallenge`, `hmacSha256`).

- [ ] **Step 6: Tests PASS**

- [ ] **Step 7: Commit**

---

### Task 7: Audit log writer (hash-chain, in-transaction)

**Files:**
- Create: `apps/api/src/db/audit.ts`
- Test: `apps/api/test/db/audit-chain.test.ts`, `immutability.test.ts`

- [ ] **Step 1: Failing test — chain integrity**
```ts
it('writeAudit increments seq, links prev_hash to last row, computes row_hash', async () => {
  const a = await writeAudit(db, { actorUserId:'u1', actorSource:'web', action:'login', recordType:'session', recordId:'s1' });
  const b = await writeAudit(db, { actorUserId:'u1', actorSource:'web', action:'page.view', recordType:'page', recordId:'/dashboard' });
  expect(a.seq).toBe(1); expect(a.prevHash).toBe(GENESIS_PREV_HASH);
  expect(b.seq).toBe(2); expect(b.prevHash).toBe(a.rowHash);
});
it('parallel writes serialize via D1 tx — no duplicate seq', async () => {
  const proms = Array.from({length:10}, (_,i) => writeAudit(db, { action:`a${i}`, actorSource:'system' }));
  const out = await Promise.all(proms);
  const seqs = out.map(o=>o.seq).sort((x,y)=>x-y);
  expect(seqs).toEqual([1,2,3,4,5,6,7,8,9,10]);
});
```

- [ ] **Step 2: Failing test — immutability**
```ts
it('UPDATE on audit_log row throws "audit_log is immutable"', async () => {
  await expect(db.run(sql`UPDATE audit_log SET action='x' WHERE seq=1`)).rejects.toThrow(/immutable/);
});
it('DELETE on audit_log row throws', async () => {
  await expect(db.run(sql`DELETE FROM audit_log WHERE seq=1`)).rejects.toThrow();
});
```

- [ ] **Step 3: Implement**
```ts
export async function writeAudit(db, payload) {
  return db.transaction(async (tx) => {
    const last = await tx.query.auditLog.findFirst({ orderBy: desc(auditLog.seq) });
    const seq = (last?.seq ?? 0) + 1;
    const prevHash = last?.rowHash ?? GENESIS_PREV_HASH;
    const ts = new Date().toISOString();
    const id = ulid();
    const fields = { id, seq, ts, prevHash, ...payload };
    const rowHash = await computeRowHash(fields);
    await tx.insert(auditLog).values({ ...fields, rowHash });
    return { ...fields, rowHash };
  });
}
```

- [ ] **Step 4: Tests PASS**

- [ ] **Step 5: Commit**

---

### Task 8: Application session JWT (idle + absolute)

**Files:**
- Create: `apps/api/src/auth/session.ts`
- Test: `apps/api/test/auth/session.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('signSession includes uid, role, iat, idle_exp, abs_exp; HS256 with JWT_SIGNING_KEY', async () => { /* ... */ });
it('verifySession returns ok=false when now > abs_exp', async () => { /* ... */ });
it('verifySession returns ok=false when now > idle_exp; renewIdle bumps idle_exp not abs_exp', async () => {
  const tok = await signSession({ uid:'u1', role:'sales' });
  await sleep(31 * 60 * 1000); // simulate via fake timers
  const r = await verifySession(tok);
  expect(r.ok).toBe(false);
  expect(r.reason).toBe('idle_timeout');
});
it('absolute exp = 2h from initial sign even after multiple renewals', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** with `jose` HS256. Token claims:
```ts
{ uid, role, sid, iat, idle_exp, abs_exp }
```
`renewIdle(token)` issues new token with `idle_exp = now + 30min`, keeps `abs_exp` unchanged.

- [ ] **Step 3: HttpOnly Secure SameSite=Strict cookie helpers**.

- [ ] **Step 4: Commit**

---

### Task 9: Cloudflare Access JWT verifier

**Files:**
- Create: `apps/api/src/auth/cf-access.ts`
- Test: `apps/api/test/auth/cf-access.test.ts`

- [ ] **Step 1: Failing test (mocked JWKS)**
```ts
it('verifyCfAccess fetches JWKS from team domain, verifies signature, checks aud', async () => { /* ... */ });
it('rejects token with wrong aud', async () => { /* ... */ });
it('rejects expired token', async () => { /* ... */ });
it('extracts email + identity_nonce + sub from valid token', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** using `jose.createRemoteJWKSet(new URL('https://${TEAM_DOMAIN}.cloudflareaccess.com/cdn-cgi/access/certs'))` with 1h cache.

- [ ] **Step 3: Implement local-dev bypass**
```ts
if (env.ENV === 'local' && req.headers.get('X-Local-Bypass') === env.LOCAL_BYPASS_TOKEN) {
  return { email: 'dev@local', sub: 'dev-sub', mfa: true };
}
```
Document in DEV.md that production must NOT set `LOCAL_BYPASS_TOKEN`.

- [ ] **Step 4: Commit**

---

### Task 10: Entra OIDC PKCE flow (post-Cloudflare Access)

**Files:**
- Create: `apps/api/src/auth/entra.ts`
- Create: `apps/api/src/routes/auth.ts`
- Test: `apps/api/test/auth/entra.test.ts`, `routes/auth.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('GET /auth/login: requires CF Access JWT, generates PKCE pair, sets state cookie, redirects to /authorize with code_challenge', async () => { /* ... */ });
it('GET /auth/callback: validates state, exchanges code for tokens via PKCE, verifies ID token JWKS + tenant_id + aud + amr contains "mfa", upserts users row, issues app session, writes audit "login.success"', async () => { /* ... */ });
it('callback rejects when amr lacks "mfa"', async () => { /* ... */ });
it('callback rejects when ID token tid != ENTRA_TENANT_ID', async () => { /* ... */ });
it('callback fails closed when CF Access email != ID token preferred_username', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** discovery (`/.well-known/openid-configuration`), token exchange, JWKS verify, claim checks. **CF Access email must match Entra preferred_username** — if mismatch, hard fail (defense in depth).

- [ ] **Step 3: User upsert** by `entra_oid` first, fallback by email; preserve `role` and `disabled` if existing.

- [ ] **Step 4: GET /auth/me** returns `{ id, email, displayName, role, teamId }` from session.

- [ ] **Step 5: POST /auth/logout** revokes session + writes audit.

- [ ] **Step 6: Tests PASS**

- [ ] **Step 7: Commit**

---

### Task 11: Auth middleware + RBAC + SoD helpers

**Files:**
- Create: `apps/api/src/auth/middleware.ts`, `rbac.ts`
- Test: `apps/api/test/auth/middleware.test.ts`, `rbac.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('requireSession: rejects missing/expired session 401', async () => { /* ... */ });
it('requireSession: bumps idle_exp on success and re-issues cookie', async () => { /* ... */ });
it('requireRole(["admin","qa_manager"]): rejects "sales" with 403, accepts "admin"', async () => { /* ... */ });
it('requireSoD enforces creator !== approver: throws 409 when creatorId === approverId', async () => {
  expect(() => requireSoD({ rule:'SoD-1', creatorId:'u1', approverId:'u1' })).toThrow(/separation of duties/);
});
it('requireDisabledFalse: throws 403 if user.disabled', async () => { /* ... */ });
```

- [ ] **Step 2: Implement**
```ts
export const requireRole = (allowed: Role[]) => async (c, next) => {
  const u = c.var.user;
  if (!allowed.includes(u.role)) throw new AppError('forbidden', 403);
  await next();
};
export function requireSoD({ rule, creatorId, approverId }) {
  if (creatorId === approverId) throw new AppError(`separation of duties violation (${rule})`, 409);
}
```

- [ ] **Step 3: Tests PASS**

- [ ] **Step 4: Commit**

---

### Task 12: Re-auth endpoint (for e-sig)

**Files:**
- Create: `apps/api/src/auth/reauth.ts`
- Modify: `apps/api/src/routes/auth.ts` (POST /auth/reauth)
- Test: `apps/api/test/auth/reauth.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /auth/reauth/init returns ROPC=false (we do not use ROPC) + step_up_url for MFA challenge', async () => { /* ... */ });
it('POST /auth/reauth/complete with MFA-elevated CF Access token issues a 2-min reauth_token tied to current session', async () => { /* ... */ });
it('reauth_token has aud="esig", linked to session id', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** — re-auth uses Cloudflare Access "purpose justification + step-up MFA" flow:
  1. Client calls `/auth/reauth/init` → server returns `step_up_url` pointing to Cloudflare Access force-MFA URL
  2. Client opens popup → user re-MFA's → CF Access issues fresh JWT with `iat` ≤ 60s
  3. Client calls `/auth/reauth/complete` with new CF Access token → server verifies freshness + matches session user → issues 2-min `reauth_token` (HS256 JWT, aud=esig)

- [ ] **Step 3: Reauth_token used as proof in next 120s for `/esig/sign`**.

- [ ] **Step 4: Tests PASS**

- [ ] **Step 5: Commit**

---

### Task 13: E-signature signing route

**Files:**
- Create: `apps/api/src/routes/esig.ts`
- Test: `apps/api/test/routes/esig.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /esig/sign requires reauth_token; rejects without with 401', async () => { /* ... */ });
it('signs record: writes audit_log row + e_signatures row in same tx; record_hash = sha256(canonical(record_payload))', async () => {
  const recordPayload = { id:'demo-1', kind:'phase0_demo', text:'I confirm reading the policy' };
  const res = await app.request('/esig/sign', {
    method:'POST', headers:{ ...auth, 'X-Reauth-Token': reauthToken },
    body: JSON.stringify({
      record_type:'phase0_demo', record_id:'demo-1',
      record_payload: recordPayload, meaning:'authorship', reason:'Phase 0 demo'
    })
  });
  expect(res.status).toBe(201);
  const body = await res.json();
  expect(body.id).toMatch(/^[0-9A-HJKMNP-TV-Z]{26}$/); // ULID
  // audit log seq incremented:
  const [aud] = await db.select().from(auditLog).where(eq(auditLog.id, body.audit_id));
  expect(aud.action).toBe('esig.signed');
});
it('record_hash is deterministic for same payload', async () => { /* ... */ });
it('user_name_snapshot frozen at sign time even if user.displayName changes later', async () => { /* ... */ });
it('GET /esig/:id returns full sig with signer name + meaning + record_hash', async () => { /* ... */ });
```

- [ ] **Step 2: Implement**
```ts
esigRoute.post('/sign', requireSession, requireReauthToken(), async (c) => {
  const body = await c.req.json();
  const u = c.var.user;
  const recordHash = await sha256Hex(canonicalJson(body.record_payload));
  return await db.transaction(async (tx) => {
    const audit = await writeAudit(tx, {
      actorUserId: u.id, actorSource:'web', action:'esig.signed',
      recordType: body.record_type, recordId: body.record_id,
      metadataJson: JSON.stringify({ meaning: body.meaning, reason: body.reason, record_hash: recordHash }),
    });
    const sig = {
      id: ulid(), userId: u.id,
      userNameSnapshot: u.displayName, userRoleSnapshot: u.role,
      recordType: body.record_type, recordId: body.record_id,
      meaning: body.meaning, reason: body.reason ?? null,
      recordHash, signedAt: new Date().toISOString(),
      reauthMethod: c.var.reauthMethod, ipAddress: c.req.header('cf-connecting-ip'),
      userAgent: c.req.header('user-agent'), prevAuditHash: audit.prevHash, auditId: audit.id,
    };
    await tx.insert(eSignatures).values(sig);
    return c.json({ ...sig, audit_id: audit.id }, 201);
  });
});
```

- [ ] **Step 3: Reauth_token consumed once** (KV nonce blacklist).

- [ ] **Step 4: Tests PASS**

- [ ] **Step 5: Commit**

---

### Task 14: Dev seed (organization + 7-role demo users)

**Files:**
- Create: `apps/api/src/seed.ts`
- Test: `apps/api/test/seed.test.ts`

- [ ] **Step 1: Failing test** — `bun src/seed.ts` populates `organization` (1 row) + 7 demo users covering all roles + 2 teams.

- [ ] **Step 2: Implement**
```ts
await db.insert(organization).values({ id:1, orgName:'Demo Medical Co.', entraTenantId:'demo-tenant', retentionYears:7, /* ... */ });
const users = [
  { email:'admin@demo', displayName:'系統管理員', role:'admin' },
  { email:'qa@demo', displayName:'品保主管', role:'qa_manager' },
  { email:'smgr@demo', displayName:'業務主管', role:'sales_manager' },
  { email:'sales@demo', displayName:'業務員', role:'sales' },
  { email:'ops@demo', displayName:'生產員', role:'ops' },
  { email:'acct@demo', displayName:'會計', role:'accountant' },
  { email:'audit@demo', displayName:'稽核員', role:'viewer' },
];
for (const u of users) await db.insert(usersTable).values({ id: ulid(), ...u, /* ... */ });
```

- [ ] **Step 3: Genesis audit row** — write `seq=1, action='system.bootstrap', prev_hash=GENESIS`.

- [ ] **Step 4: Commit**

---

### Task 15: i18n (zh-TW + en)

**Files:**
- Create: `apps/web/src/i18n/index.ts`, `zh-TW.json`, `en.json`
- Test: `apps/web/test/i18n.test.ts`

- [ ] **Step 1: Failing test** — `t('app.title')` resolves both locales; missing key throws in dev, returns key in prod.

- [ ] **Step 2: Implement** light wrapper (no i18next dep — small JSON map + Context).

- [ ] **Step 3: Seed strings** for: app title, login, logout, role names, e-sig dialog labels (姓名/意涵/簽章時間/原因/我已閱讀並同意/取消).

- [ ] **Step 4: Commit**

---

### Task 16: UI shell + Login + E-sig dialog + Demo button

**Files:**
- Create: `apps/web/src/routes/__root.tsx`, `index.tsx`, `login.tsx`
- Create: `apps/web/src/components/shell/AppShell.tsx`, `Sidebar.tsx`, `TopBar.tsx`
- Create: `apps/web/src/components/esig/EsigDialog.tsx`, `EsigDemoButton.tsx`
- Create: `apps/web/src/lib/api.ts`, `auth.ts`, `esig.ts`

- [ ] **Step 1: Implement TopBar** showing `displayName + role badge + logout`.

- [ ] **Step 2: Implement EsigDialog** — props: `recordType`, `recordId`, `recordPayload`, `meaning`, `onSigned`. Flow: shows record summary → reason field → step-up MFA popup → calls `/esig/sign` → renders signature manifestation snippet.

- [ ] **Step 3: Implement Login flow** — `/login` → button "M365 SSO" → redirect to `/api/auth/login` → callback handled by API → cookie set → redirect to `/`.

- [ ] **Step 4: Dashboard page** shows: org name, current user info, "Phase 0 Demo: 簽署一筆 demo 紀錄" button → opens EsigDialog → on success shows signed record + "查看 audit 鏈" link.

- [ ] **Step 5: Audit chain viewer** (admin/qa_manager/viewer only) — fetch `/audit?limit=20` → render table with seq + action + actor + ts + row_hash[:8].

- [ ] **Step 6: Commit**

---

### Task 17: E2E — login + e-sig + audit chain verify

**Files:**
- Create: `apps/web/tests/e2e/auth.spec.ts`, `esig-demo.spec.ts`, `audit-chain.spec.ts`

- [ ] **Step 1: Auth E2E**
```ts
test('M365 mock SSO login → dashboard shows display name + role', async ({ page }) => {
  await mockCfAccess(page, { email:'qa@demo' });
  await mockEntra(page, { oid:'qa-oid', tid: ENTRA_TENANT_ID, amr:['mfa'], preferred_username:'qa@demo' });
  await page.goto('/login');
  await page.click('button:has-text("M365 SSO")');
  await expect(page).toHaveURL('/');
  await expect(page.getByTestId('user-display')).toHaveText('品保主管');
  await expect(page.getByTestId('user-role')).toHaveText('qa_manager');
});
```

- [ ] **Step 2: E-sig demo E2E**
```ts
test('Phase 0 demo: sign and verify audit row created', async ({ page }) => {
  await loginAs(page, 'qa_manager');
  await page.click('button:has-text("簽署一筆 demo")');
  await page.fill('textarea[name=reason]', 'Phase 0 verification');
  await mockReauth(page);  // step-up MFA mock
  await page.click('button:has-text("我已閱讀並同意")');
  await expect(page.getByTestId('signed-confirmation')).toBeVisible();
  await expect(page.getByTestId('signed-name')).toHaveText('品保主管');
  await expect(page.getByTestId('signed-meaning')).toHaveText('authorship');
});
```

- [ ] **Step 3: Audit chain integrity E2E**
```ts
test('audit chain verifier returns ok=true after several events', async ({ page, request }) => {
  await loginAs(page, 'admin');
  await page.click('text=系統 → 稽核軌跡完整性檢查');
  await expect(page.getByTestId('chain-status')).toHaveText('OK');
  await expect(page.getByTestId('chain-rows')).toContainText(/[\d]+ 筆/);
});
```

- [ ] **Step 4: Tests PASS**

- [ ] **Step 5: Commit**

---

### Task 18: Phase 0 verification + checklist

**Files:**
- Create: `docs/validation/phase0-iq.md`, `phase0-oq.md`
- Modify: `README.md`

- [ ] **Step 1: IQ checklist** (Installation Qualification)
- [ ] D1 `orderflow` 建立 + migrations applied
- [ ] R2 buckets `orderflow-docs` + `orderflow-immutable` 建立（Phase 4 才實際使用，先建）
- [ ] Wrangler secrets set: `JWT_SIGNING_KEY`, `AUDIT_HASH_PEPPER`, `ENTRA_CLIENT_SECRET`
- [ ] Cloudflare Access app created, policy: M365 + MFA required
- [ ] Entra app registered, redirect URI matches
- [ ] DNS: `orderflow.internal.<company>.com` resolves to Pages

- [ ] **Step 2: OQ checklist** (Operational Qualification)
- [ ] Login through real M365 (not mock) → audit_log row 寫入 with `action='login.success'` + valid hash chain
- [ ] Try expired session → 401 + redirect to /login
- [ ] Try non-MFA session (mock amr=[]) → rejected
- [ ] Sign demo record → e_signatures row created + audit_log paired row
- [ ] Attempt UPDATE audit_log → trigger blocks
- [ ] Attempt DELETE e_signatures → trigger blocks
- [ ] Run `bun src/scripts/verify-chain.ts` → reports OK + N rows
- [ ] Disable a user (UPDATE users SET disabled=1) → next login blocked
- [ ] requireSoD throws 409 when creatorId===approverId

- [ ] **Step 3: Tag**
```bash
git tag -a v0.0.1-phase0 -m "Phase 0 foundation complete: SSO + audit chain + e-sig framework"
```

- [ ] **Step 4: Final commit**
```bash
git add docs/validation README.md
git commit -m "docs(phase0): IQ/OQ checklists + verification"
```

---

## Phase 0 Done When

1. All 18 tasks ✅
2. IQ + OQ checklists all green
3. Hash-chain verifier validates entire `audit_log` from genesis
4. Demo e-sig flow works end-to-end with real M365 + Cloudflare Access
5. No `tenant_id` columns anywhere (grep `tenant_id` in `apps/api/src` returns nothing)
6. Phase 1 can begin (orders + customers + Lot/UDI)
