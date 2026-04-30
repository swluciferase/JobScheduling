# OrderFlow Phase 3: MCP Server + Multi-Tenant SaaS Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expose OrderFlow as a Model Context Protocol (MCP) server so users can query and control orders from Microsoft Teams Copilot (or any MCP client) using natural language; and turn the system into a multi-tenant SaaS with self-service signup, public deployment, and tenant-isolated billing-ready foundations.

**Architecture:** Add a separate `apps/mcp` Worker that hosts the MCP HTTP+SSE transport, authenticated via OAuth 2.0 Device Flow (per spec §13.mcp-auth). Reuse all Phase 1/2 service-layer code via shared `packages/shared`. Tools translate to existing API calls; results truncate to LLM-safe sizes. Add multi-tenant signup (org create + verify), tenant settings UI, per-tenant Entra app registration support. Move from localhost to public Cloudflare Pages + Workers deployment with custom domain `orderflow.<tenantSlug>.example.com` (or path-based).

**Tech Stack:** Same as Phase 1/2 + `@modelcontextprotocol/sdk` (TypeScript SDK), Bot Framework Service for Teams Copilot integration, Stripe (deferred to v0.4 — Phase 3 only sets up the data model + admin UI for plans, no checkout). Public deployment via wrangler `routes` + Cloudflare custom domain.

**Phase boundary:** Ends when (a) a Teams Copilot user can ask "What's the status of order ORD-20260430-0001?" and get an Adaptive Card response after granting OAuth Device Flow consent once; (b) a new organization can sign up at `signup.orderflow.example.com`, verify email, configure their Entra tenant ID, and onboard their first user; (c) the platform is publicly deployed with HTTPS + custom domain.

---

## Reference Spec

Cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §13 MCP server architecture, 8 tools, OAuth Device Flow
- §3 Multi-tenancy (signup flow + per-tenant Entra config)
- §14 Deployment topology (public deploy)
- §15 Future: billing data model

---

## File Structure

```
orderflow/
├── apps/
│   ├── mcp/                              # NEW Worker
│   │   ├── package.json
│   │   ├── wrangler.toml
│   │   ├── src/
│   │   │   ├── index.ts                  # MCP server entry
│   │   │   ├── transport.ts              # HTTP + SSE transport
│   │   │   ├── auth/
│   │   │   │   ├── device-flow.ts
│   │   │   │   ├── token-store.ts        # KV-backed
│   │   │   │   └── verify.ts
│   │   │   ├── tools/
│   │   │   │   ├── list-orders.ts
│   │   │   │   ├── get-order.ts
│   │   │   │   ├── get-order-alerts.ts
│   │   │   │   ├── search-customer.ts
│   │   │   │   ├── advance-stage.ts
│   │   │   │   ├── add-note.ts
│   │   │   │   ├── get-logistics.ts
│   │   │   │   ├── upload-document.ts
│   │   │   │   └── index.ts
│   │   │   ├── middleware/
│   │   │   │   ├── rate-limit.ts
│   │   │   │   └── audit.ts
│   │   │   └── adapters/
│   │   │       └── core-api.ts            # calls apps/api via internal token
│   │   └── teams/
│   │       ├── manifest.json              # Teams app manifest
│   │       └── icons/
│   └── api/
│       └── src/
│           ├── routes/
│           │   ├── signup.ts
│           │   ├── tenants.ts             # admin self-service settings
│           │   ├── plans.ts               # billing plans (data only)
│           │   └── invitations.ts
│           └── services/
│               ├── tenant-provision.ts
│               └── email-verify.ts
└── apps/web/src/
    ├── routes/
    │   ├── signup/
    │   │   ├── index.tsx                  # public org create
    │   │   ├── verify.tsx
    │   │   └── onboarding.tsx
    │   ├── settings/
    │   │   ├── tenant.tsx
    │   │   ├── users.tsx
    │   │   ├── teams.tsx
    │   │   └── integrations/
    │   │       ├── entra.tsx
    │   │       ├── teams.tsx
    │   │       └── slack.tsx
    │   └── invite/
    │       └── $token.tsx
    └── features/
        └── settings/
            └── EntraConfigForm.tsx
```

---

## Task List Overview

Week 1: Tasks 1–5 (MCP server scaffold, transport, OAuth Device Flow)
Week 2: Tasks 6–11 (8 MCP tools)
Week 3: Tasks 12–14 (audit + rate limit + Teams app manifest + Copilot integration test)
Week 4: Tasks 15–20 (multi-tenant signup, public deployment, custom domain, billing data model)

---

### Task 1: MCP Worker scaffold

**Files:**
- Create: `apps/mcp/package.json`, `wrangler.toml`, `tsconfig.json`
- Create: `apps/mcp/src/index.ts`
- Modify: root `package.json` (add `apps/mcp` workspace)
- Test: `apps/mcp/test/health.test.ts`

- [ ] **Step 1: Failing health test**
```ts
import { unstable_dev } from 'wrangler';
it('GET /health returns ok with version', async () => {
  const worker = await unstable_dev('apps/mcp/src/index.ts', { experimental:{ disableExperimentalWarning:true } });
  const res = await worker.fetch('/health');
  expect(await res.json()).toMatchObject({ ok:true, server:'orderflow-mcp' });
  await worker.stop();
});
```

- [ ] **Step 2: Implement Hono Worker** with `/health`, `/.well-known/oauth-authorization-server`, `/mcp` (POST + SSE), `/oauth/device`, `/oauth/token`, `/oauth/verify`.

- [ ] **Step 3: Bind D1 + KV + Queues + R2** in wrangler.toml (read-only KV for token cache).

- [ ] **Step 4: Test PASS**

- [ ] **Step 5: Commit**
```bash
git commit -am "feat(mcp): worker scaffold + health endpoint"
```

---

### Task 2: MCP transport (HTTP + SSE)

**Files:**
- Create: `apps/mcp/src/transport.ts`
- Modify: `apps/mcp/src/index.ts`
- Test: `apps/mcp/test/transport.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /mcp with initialize returns serverCapabilities, tools/list returns 8 tools', async () => {
  const init = await client.send({ jsonrpc:'2.0', id:1, method:'initialize', params:{ protocolVersion:'2024-11-05' } });
  expect(init.result.capabilities.tools).toBeDefined();
  const tools = await client.send({ jsonrpc:'2.0', id:2, method:'tools/list', params:{} });
  expect(tools.result.tools.length).toBe(8);
});
it('SSE /mcp opens long-poll, server can push messages with same session', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** using `@modelcontextprotocol/sdk` `Server` + custom HTTP transport. Map JSON-RPC over POST; SSE for server→client streaming. Session ID via `Mcp-Session-Id` header backed by KV.

- [ ] **Step 3: Tests PASS** (tools list will be 0 until Task 6+ — temporarily expect 0, update when tools land)

- [ ] **Step 4: Commit**

---

### Task 3: OAuth 2.0 Device Flow — discovery + device endpoint

**Files:**
- Create: `apps/mcp/src/auth/device-flow.ts`
- Modify: `apps/mcp/src/index.ts`
- Create: `apps/mcp/src/db/migrations/0001_oauth.sql` + table `oauth_clients (client_id, client_name, redirect_uris_json, allowed_scopes, created_at)`, `oauth_device_codes (device_code, user_code, client_id, scope, status, user_id, tenant_id, expires_at, created_at)`, `oauth_tokens (token, refresh_token, client_id, user_id, tenant_id, scope, expires_at, revoked_at)`
- Test: `apps/mcp/test/auth/device-flow.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('GET /.well-known/oauth-authorization-server returns RFC8414 metadata with device_authorization_endpoint', async () => { /* ... */ });
it('POST /oauth/device returns user_code (8 chars), device_code, verification_uri, expires_in=600, interval=5', async () => { /* ... */ });
it('POST /oauth/token with grant_type=device_code: returns 400 authorization_pending until user verifies, then access+refresh token', async () => { /* ... */ });
it('user_code is single-use, expired after 10min returns expired_token', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** RFC 8628. `user_code` is 8 chars from `BCDFGHJKLMNPQRSTVWXYZ23456789` (omit confusable chars). Token = `mcp_<base64url 32B>`.

- [ ] **Step 3: Commit**

---

### Task 4: OAuth verify UI

**Files:**
- Create: `apps/web/src/routes/oauth/verify.tsx`
- Create: `apps/mcp/src/auth/verify.ts` (POST /oauth/verify backend)
- Test: `apps/web/e2e/oauth-verify.spec.ts`

- [ ] **Step 1: Failing E2E**
```ts
test('user logs into web app, opens /oauth/verify, enters user_code from MCP client, approves scope → device polling endpoint returns access_token', async ({ page }) => {
  // start device flow externally → get user_code "WXYZ-2345"
  await loginAs(page, 'sales');
  await page.goto('/oauth/verify');
  await page.fill('input[name=user_code]', 'WXYZ2345');
  await page.click('button:has-text("下一步")');
  await expect(page.getByText('OrderFlow MCP Client')).toBeVisible();
  await expect(page.getByText('讀取您的訂單')).toBeVisible();
  await page.click('button:has-text("授權")');
  // poll device_code endpoint → expect 200 with access_token
});
```

- [ ] **Step 2: Implement** — verify page reads `user_code`, looks up `oauth_device_codes`, shows scope list (i18n), buttons approve/reject; on approve, set `status='approved'` + `user_id` + `tenant_id`, redirect to success page.

- [ ] **Step 3: Token endpoint polling returns the issued token after approval.**

- [ ] **Step 4: Commit**

---

### Task 5: MCP auth middleware + token store

**Files:**
- Create: `apps/mcp/src/auth/token-store.ts`
- Modify: `apps/mcp/src/transport.ts` (add bearer auth on /mcp)
- Test: `apps/mcp/test/auth/middleware.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('requests to /mcp without bearer return 401 with WWW-Authenticate', async () => { /* ... */ });
it('valid token sets ctx.user_id/tenant_id, expired token returns invalid_token', async () => { /* ... */ });
it('revoked token (oauth_tokens.revoked_at IS NOT NULL) returns invalid_token', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** — read `Authorization: Bearer ...`; KV lookup with 60s cache, fallback to D1. Cache `revoked` state too.

- [ ] **Step 3: Commit**

---

### Task 6: Tools — list_orders + get_order + search_customer

**Files:**
- Create: `apps/mcp/src/tools/list-orders.ts`, `get-order.ts`, `search-customer.ts`, `index.ts`
- Create: `apps/mcp/src/adapters/core-api.ts` (typed wrapper around apps/api fetch)
- Test: `apps/mcp/test/tools/read-tools.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('list_orders truncates to 20 rows max + provides next_cursor', async () => {
  const res = await callTool('list_orders', { status:'active' });
  expect(res.content[0].text).toMatch(/"orders":\[/);
  expect(JSON.parse(res.content[0].text).orders.length).toBeLessThanOrEqual(20);
});
it('get_order respects role redaction (token issued for sales does not return total_amount when role=viewer)', async () => { /* ... */ });
it('search_customer fuzzy matches name + tax_id', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** each tool with Zod input schema, return `content: [{ type:'text', text: JSON.stringify(...) }]`. Truncate `notes` to 500 chars + `... (truncated)`.

- [ ] **Step 3: Adapter** uses internal token `X-MCP-Internal: <secret>` to call apps/api with the user's identity propagated via `X-MCP-User-Id` + `X-MCP-Tenant-Id`. apps/api must validate the secret and trust the headers.

- [ ] **Step 4: Modify apps/api auth middleware** to accept internal MCP calls when secret valid.

- [ ] **Step 5: Commit**

---

### Task 7: Tools — get_order_alerts + get_logistics

**Files:**
- Create: `apps/mcp/src/tools/get-order-alerts.ts`, `get-logistics.ts`
- Test: `apps/mcp/test/tools/read-tools.test.ts` (extend)

- [ ] **Step 1: Failing test** — `get_order_alerts` returns open alerts with severity + escalation_level; `get_logistics` returns last 20 events.

- [ ] **Step 2: Implement**

- [ ] **Step 3: Commit**

---

### Task 8: Tool — advance_stage (write)

**Files:**
- Create: `apps/mcp/src/tools/advance-stage.ts`
- Test: `apps/mcp/test/tools/write-tools.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('advance_stage requires "orders:write" scope on token, otherwise insufficient_scope error', async () => { /* ... */ });
it('advance_stage call with note writes audit_log entry tagged actor=mcp:userId', async () => { /* ... */ });
it('advance_stage refused when role=viewer even with write scope', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** — validate `orders:write` scope present; call apps/api `/orders/:id/stages/advance` with user identity; return summary `{ ok:true, prev_stage, new_stage }`.

- [ ] **Step 3: Commit**

---

### Task 9: Tool — add_note + upload_document

**Files:**
- Create: `apps/mcp/src/tools/add-note.ts`, `upload-document.ts`

- [ ] **Step 1: Failing test** — `add_note` writes `order_stages.meta_json.notes[]` entry; `upload_document` accepts `{ doc_type, file_name, base64_content }` ≤ 5MB, returns `{ document_id, version }`.

- [ ] **Step 2: Implement** — for upload, decode base64, validate mime/size, presign+PUT internally, commit document row.

- [ ] **Step 3: Commit**

---

### Task 10: Audit logging for MCP

**Files:**
- Create: `apps/mcp/src/middleware/audit.ts`
- Modify: each tool to wrap with audit
- Modify: `audit_log` schema — add `source` column ∈ `{web, api, mcp}`; migration `0006_audit_source.sql`
- Test: `apps/mcp/test/audit.test.ts`

- [ ] **Step 1: Failing test** — every tool call writes `audit_log` row with `source='mcp'`, `actor=user_id`, `metadata={ tool_name, args_hash }` (do not log full args — PII).

- [ ] **Step 2: Implement**

- [ ] **Step 3: Commit**

---

### Task 11: Rate limiting on MCP

**Files:**
- Create: `apps/mcp/src/middleware/rate-limit.ts`
- Test: `apps/mcp/test/rate-limit.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('per-token: 60 calls/min, 1000 calls/hour, returns 429 with Retry-After', async () => { /* ... */ });
it('write tools share separate budget: 20 writes/min', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** using Cloudflare Rate Limiting binding (preferred) or Durable Object counter as fallback. Keys: `mcp:rl:read:<token>:<minute>` and `mcp:rl:write:<token>:<minute>`.

- [ ] **Step 3: Commit**

---

### Task 12: Teams app manifest + Adaptive Card responses

**Files:**
- Create: `apps/mcp/teams/manifest.json`, `icons/color.png`, `icons/outline.png`
- Modify: each tool — return `content[1]` with Adaptive Card JSON when invoked from Teams (detect via `_meta.client = 'teams'`)
- Test: `apps/mcp/test/teams-card.test.ts`

- [ ] **Step 1: Failing test** — `get_order` from Teams returns Adaptive Card v1.5 with order header, stage progress, key facts, "查看詳情" action linking to web app.

- [ ] **Step 2: Author manifest**
```json
{
  "manifestVersion": "1.17",
  "id": "<uuid>",
  "name": { "short": "OrderFlow", "full": "OrderFlow Order Tracking" },
  "developer": { ... },
  "copilotAgents": {
    "declarativeAgents": [{
      "id": "orderflow-agent",
      "file": "agent.json"
    }]
  }
}
```

- [ ] **Step 3: Implement** card builder util.

- [ ] **Step 4: Commit**

---

### Task 13: Copilot integration smoke (manual + scripted)

**Files:**
- Create: `apps/mcp/scripts/test-copilot.ts` (uses `@microsoft/teams-cli` or REST to validate manifest)
- Modify: `README.md`

- [ ] **Step 1: Manual checklist (cannot fully automate)**
- [ ] Sideload Teams app with manifest in dev tenant
- [ ] Open Copilot, ask "@OrderFlow show order ORD-20260430-0001"
- [ ] Confirm Adaptive Card renders
- [ ] Ask "what alerts on this order?" → calls `get_order_alerts`
- [ ] Ask "advance to next stage" → triggers consent re-check or scope insufficient error if read-only

- [ ] **Step 2: Scripted: validate manifest with `teamsapp validate ./apps/mcp/teams/manifest.json`**

- [ ] **Step 3: Commit checklist + script**

---

### Task 14: Public OAuth 2.0 client registration UI

**Files:**
- Create: `apps/web/src/routes/settings/integrations/mcp-clients.tsx` (admin only)
- Modify: `apps/api/src/routes/oauth-clients.ts` (CRUD)
- Test: `apps/api/test/routes/oauth-clients.test.ts`

- [ ] **Step 1: Failing test** — admin can create client, generates `client_id` + `client_secret` (shown once), can revoke; non-admin 403.

- [ ] **Step 2: Implement** — store hashed secret; allow rotate.

- [ ] **Step 3: Commit**

---

### Task 15: Public signup — org create + email verify

**Files:**
- Create: `apps/api/src/routes/signup.ts`
- Create: `apps/api/src/services/tenant-provision.ts`, `email-verify.ts`
- Create: `apps/api/src/db/migrations/0007_signup.sql` — table `pending_signups (id, email, org_name, slug, verify_token, expires_at, created_at)`
- Create: `apps/web/src/routes/signup/index.tsx`, `verify.tsx`, `onboarding.tsx`
- Test: `apps/api/test/routes/signup.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /signup with org_name + email + password creates pending_signup, sends verify email, slug uniqueness enforced', async () => { /* ... */ });
it('GET /signup/verify?token=... activates tenant, creates first admin user, expires token', async () => { /* ... */ });
it('rate limits signup to 3/hour per IP', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** signup → email via Resend → verify link → create `tenants` row + first `users` row with role=admin → onboarding wizard.

- [ ] **Step 3: Reserved slugs** (`api`, `app`, `signup`, `admin`, `mcp`, `auth`, `oauth`, `www`).

- [ ] **Step 4: Commit**

---

### Task 16: Tenant settings UI

**Files:**
- Create: `apps/web/src/routes/settings/tenant.tsx`, `users.tsx`, `teams.tsx`
- Create: `apps/web/src/routes/settings/integrations/entra.tsx`, `teams.tsx`, `slack.tsx`
- Modify: `apps/api/src/routes/tenants.ts` — admin can update tenant_name, brand_json, default_stage_template_id, notification webhooks

- [ ] **Step 1: Failing E2E** — admin updates tenant name + Slack webhook URL → next alert sends to new webhook.

- [ ] **Step 2: Implement** with feature flags per tenant for email/teams/slack channels.

- [ ] **Step 3: Commit**

---

### Task 17: Per-tenant Entra config + invite flow

**Files:**
- Modify: `apps/api/src/routes/auth.ts` — accept `tenant_slug` parameter on `/login` to look up tenant Entra config
- Create: `apps/api/src/routes/invitations.ts`
- Create: `apps/web/src/routes/invite/$token.tsx`
- Test: `apps/api/test/auth/multi-tenant.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('login route resolves tenant_slug from subdomain or query, uses tenant Entra app', async () => { /* ... */ });
it('admin invites user, email contains link with token; clicking creates user in same tenant with chosen role', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** — `tenants.entra_config_json = { tenant_id, client_id, client_secret_ref, redirect_uri }`. Phase 0 single-tenant config moves into here.

- [ ] **Step 3: Invite token = 32B random in `invitations(token, tenant_id, email, role, team_id, expires_at, used_at)`.**

- [ ] **Step 4: Commit**

---

### Task 18: Public deployment — Cloudflare Pages + Workers + custom domain

**Files:**
- Modify: `apps/web/wrangler.toml` (or `pages.toml`)
- Modify: `apps/api/wrangler.toml` (add `routes` for `api.orderflow.example.com/*`)
- Modify: `apps/mcp/wrangler.toml` (`mcp.orderflow.example.com/*`)
- Create: `docs/DEPLOYMENT.md`

- [ ] **Step 1: Set up Cloudflare zone for `orderflow.example.com`** (manual via dashboard) + add CNAME records.

- [ ] **Step 2: Configure Wrangler routes**
```toml
# apps/api/wrangler.toml
routes = [
  { pattern = "api.orderflow.example.com/*", zone_name = "orderflow.example.com" }
]
```

- [ ] **Step 3: Deploy**
```bash
cd apps/web && bun run build && bunx wrangler pages deploy dist --project-name orderflow-web
cd apps/api && bunx wrangler deploy
cd apps/mcp && bunx wrangler deploy
```

- [ ] **Step 4: Smoke test public URLs** — `curl https://api.orderflow.example.com/health`, navigate to web, MCP `/.well-known/...`.

- [ ] **Step 5: Set production secrets** (Resend API key, VAPID keys, Entra secrets, MCP internal secret, R2 keys).

- [ ] **Step 6: Document deployment in DEPLOYMENT.md.**

- [ ] **Step 7: Commit**

---

### Task 19: Billing data model (no checkout — foundation only)

**Files:**
- Create: `apps/api/src/db/migrations/0008_billing.sql`
  - `plans (id, code, name, monthly_price_cents, included_users, included_orders_per_month, features_json)`
  - `tenants` add `plan_id`, `billing_email`, `subscription_status` ∈ `{trial, active, past_due, canceled}`
  - `usage_counters (id, tenant_id, metric, value, period_start, period_end)`
- Create: `apps/api/src/routes/plans.ts` (GET list, POST admin assign)
- Create: `apps/web/src/routes/settings/plan.tsx`
- Test: `apps/api/test/routes/plans.test.ts`

- [ ] **Step 1: Failing test** — admin can view plans, see current tenant plan; usage counters increment on order create / user invite.

- [ ] **Step 2: Implement** — built-in plans: Trial (free, 14 days), Starter ($49/mo, 5 users, 100 orders), Pro ($199/mo, 25 users, 1000 orders).

- [ ] **Step 3: Enforce limits softly** — show warning at 80%, hard block at 100% with upgrade CTA. Defer Stripe to v0.4.

- [ ] **Step 4: Commit**

---

### Task 20: Phase 3 verification + final smoke

**Files:**
- Create: `apps/web/e2e/phase3-smoke.spec.ts`
- Modify: `README.md`

- [ ] **Step 1: Smoke E2E**
```ts
test('phase 3 happy path: visitor signs up new org → verifies email → invites user → user gets order via MCP from Teams', async ({ page }) => { /* ... */ });
test('MCP client lifecycle: register client → device flow → list_orders → advance_stage → audit shows mcp source', async ({ page }) => { /* ... */ });
```

- [ ] **Step 2: Manual checklist**
- [ ] Sign up at public URL works end-to-end
- [ ] Per-tenant Entra config: two tenants, two different Entra app IDs, both log in correctly
- [ ] Teams Copilot invocation returns Adaptive Card
- [ ] MCP rate limits enforce (verify 429 response)
- [ ] Audit log records `source=mcp` for tool calls
- [ ] Tenant on Trial plan blocked from creating 101st order; admin upgrade flow shows correct plan options
- [ ] Production secrets all set; no secrets in repo
- [ ] HTTPS works on api/mcp/web subdomains
- [ ] Web push works in production (different VAPID keys vs dev)

- [ ] **Step 3: Tag**
```bash
git tag -a v0.3.0-phase3 -m "Phase 3 MCP + multi-tenant SaaS complete"
```

- [ ] **Step 4: Final commit**

---

## Phase 3 Done When

1. All 20 tasks ✅
2. Smoke E2E passes against public deployment
3. Manual checklist all confirmed
4. MCP server passes `mcp inspector` validation
5. Teams Copilot agent demo works end-to-end (Card rendered, write tool consent works)
6. Self-service signup: a fresh email creates a usable tenant in <5 min
7. Multi-tenant isolation re-audited: query `audit_log` for any cross-tenant data access; expect 0 rows
8. Documentation: README + DEPLOYMENT.md + MCP integration guide complete
9. Stripe checkout flow scoped for v0.4 (separate plan)

---

## Post-Phase-3 Roadmap (out of scope)

- v0.4: Stripe checkout + subscription lifecycle webhooks
- v0.5: Customer portal (limited read-only access for end customers to track their own orders)
- v0.6: Advanced reports (cohort analysis, lead time distribution, supplier performance)
- v0.7: Inventory module (deduct on packing, reorder alerts)
- v1.0: Production-grade SLOs, on-call runbook, status page, SOC 2 readiness gap analysis
