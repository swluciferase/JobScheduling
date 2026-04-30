# OrderFlow Phase 2: Mobile + Advanced Permissions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make OrderFlow usable on phones for warehouse/logistics staff (PWA + barcode scanning + SKU picking flow), and complete the advanced permission features (sales_manager team queries, accountant payment view, AR aging, high-value approval UI, SLA dashboard).

**Architecture:** Layer on top of Phase 1 MVP. Convert the web app into an installable PWA with a service worker (cache-first for shell, network-first for API). Add `BarcodeDetector`-based scanner (no offline mode, no iOS<16 fallback per spec §10.barcode). Introduce team-scoped queries gated by `team_id` linkage. Add accountant-specific routes and AR aging report. Wire end-to-end flow: ops scans SKU on packing → marks line item picked → triggers advance when all items picked.

**Tech Stack:** Same as Phase 1 + `vite-plugin-pwa` (manifest + Workbox SW), Web Push API + VAPID, BarcodeDetector (native), Workbox `precacheAndRoute`, recharts (charts already in Phase 1).

**Phase boundary:** Ends when an ops user can install the app on iOS Safari / Android Chrome, scan a SKU barcode at the packing station, mark items picked, advance to packed; a sales_manager can view their team's pipeline; an accountant can mark deposit/final paid and run AR aging; an admin can approve a high-value order through a dedicated UI; the SLA dashboard shows live breach risk. MCP and self-service signup deferred to Phase 3.

---

## Reference Spec

Cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §4 RBAC matrix (sales_manager team scope, accountant money fields, viewer read-only)
- §6 Stage `picking` for wholesale (barcode flow attaches here)
- §10 Mobile / barcode scanning (no offline; BarcodeDetector only)
- §12 Reports (AR aging, SLA dashboard)

---

## File Structure

```
orderflow/
├── apps/web/
│   ├── public/
│   │   ├── manifest.webmanifest
│   │   ├── icon-192.png
│   │   ├── icon-512.png
│   │   └── apple-touch-icon.png
│   ├── src/
│   │   ├── pwa/
│   │   │   ├── register-sw.ts
│   │   │   ├── push-subscribe.ts
│   │   │   └── install-prompt.tsx
│   │   ├── features/
│   │   │   ├── scan/
│   │   │   │   ├── BarcodeScanner.tsx
│   │   │   │   ├── ScanFeedback.tsx
│   │   │   │   └── useBarcodeDetector.ts
│   │   │   ├── picking/
│   │   │   │   ├── PickingView.tsx
│   │   │   │   ├── PickingItemRow.tsx
│   │   │   │   └── PickingComplete.tsx
│   │   │   ├── approvals/
│   │   │   │   └── ApprovalQueue.tsx
│   │   │   ├── payments/
│   │   │   │   ├── PaymentDialog.tsx
│   │   │   │   └── PaymentList.tsx
│   │   │   └── dashboards/
│   │   │       ├── SlaDashboard.tsx
│   │   │       └── AgingReport.tsx
│   │   └── routes/
│   │       ├── scan.tsx                 # entry for /scan deep-link
│   │       ├── picking/$orderId.tsx
│   │       ├── approvals/index.tsx
│   │       ├── payments/index.tsx
│   │       └── dashboards/
│   │           ├── sla.tsx
│   │           └── aging.tsx
└── apps/api/src/
    ├── routes/
    │   ├── picking.ts                   # POST /orders/:id/picking/scan
    │   ├── payments.ts                  # POST /orders/:id/payments
    │   ├── approvals.ts                 # GET /approvals (queue)
    │   ├── push.ts                      # POST /push/subscribe
    │   └── reports.ts                   # extended (aging)
    ├── services/
    │   ├── push-vapid.ts
    │   └── aging-calculator.ts
    └── adapters/
        └── push-web.ts
```

---

## Task List Overview

Week 1: Tasks 1–6 (PWA setup, manifest, SW, install, push subscribe, push adapter)
Week 2: Tasks 7–12 (barcode scan, picking flow, scan→advance integration, mobile UX polish)
Week 3: Tasks 13–18 (sales_manager team scope, accountant payments, approval UI, AR aging)
Week 4: Tasks 19–22 (SLA dashboard, role-based money field redaction, smoke E2E + verification)

---

### Task 1: PWA manifest + icons

**Files:**
- Create: `apps/web/public/manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`
- Modify: `apps/web/index.html` (add manifest link + theme-color meta)
- Test: `apps/web/test/pwa-manifest.test.ts`

- [ ] **Step 1: Failing test**
```ts
import manifest from '../public/manifest.webmanifest' with { type: 'json' };
it('manifest declares standalone, has 192/512 icons, name + short_name', () => {
  expect(manifest.display).toBe('standalone');
  expect(manifest.icons.find(i=>i.sizes==='192x192')).toBeTruthy();
  expect(manifest.icons.find(i=>i.sizes==='512x512')).toBeTruthy();
  expect(manifest.start_url).toBe('/');
});
```

- [ ] **Step 2: Author manifest**
```json
{
  "name": "OrderFlow",
  "short_name": "OrderFlow",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0f172a",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ]
}
```

- [ ] **Step 3: Generate icons** (placeholder via `sharp` script `scripts/gen-icons.ts` from one source SVG; commit PNGs).

- [ ] **Step 4: Add to index.html**
```html
<link rel="manifest" href="/manifest.webmanifest">
<meta name="theme-color" content="#0f172a">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
```

- [ ] **Step 5: Test PASS**

- [ ] **Step 6: Commit**
```bash
git commit -am "feat(pwa): manifest + icons"
```

---

### Task 2: Service worker via vite-plugin-pwa

**Files:**
- Modify: `apps/web/vite.config.ts`
- Modify: `apps/web/package.json` (add `vite-plugin-pwa`, `workbox-window`)
- Create: `apps/web/src/pwa/register-sw.ts`
- Modify: `apps/web/src/main.tsx` (call register)
- Test: `apps/web/e2e/sw.spec.ts`

- [ ] **Step 1: Failing E2E**
```ts
test('SW registers, app loads offline shell after first visit (network=offline returns 200 from cache for /, /assets/*)', async ({ page, context }) => {
  await page.goto('/');
  await page.waitForFunction(() => navigator.serviceWorker.ready);
  await context.setOffline(true);
  await page.reload();
  await expect(page.getByText('OrderFlow')).toBeVisible(); // shell from cache
});
```

- [ ] **Step 2: Configure plugin**
```ts
VitePWA({
  registerType: 'autoUpdate',
  workbox: {
    navigateFallback: '/index.html',
    runtimeCaching: [
      { urlPattern: /\/api\/.*/, handler: 'NetworkFirst', options:{ cacheName:'api', networkTimeoutSeconds:3, expiration:{ maxAgeSeconds:60 } } },
      { urlPattern: /\.(?:js|css|woff2?)$/, handler: 'CacheFirst', options:{ cacheName:'static' } },
    ],
  },
})
```

- [ ] **Step 3: register-sw.ts** wires `useRegisterSW` and shows toast on update.

- [ ] **Step 4: Test PASS**

- [ ] **Step 5: Commit**

> ⚠ Note: spec §10 says "no offline mode for data" — only the **shell** is cached. API requests fail clearly when offline. Add an offline toast in Task 3.

---

### Task 3: Install prompt + offline indicator

**Files:**
- Create: `apps/web/src/pwa/install-prompt.tsx`, `useOnlineStatus.ts`
- Modify: `apps/web/src/components/shell/AppShell.tsx`
- Test: `apps/web/test/pwa.test.tsx`

- [ ] **Step 1: Failing test** — when `navigator.onLine = false`, banner "離線中，無法操作" appears; when `beforeinstallprompt` fires, button "安裝至主畫面" appears.

- [ ] **Step 2: Implement** install button that calls `deferredPrompt.prompt()`; show iOS-specific instructions when no event fires (UA detect).

- [ ] **Step 3: Commit**

---

### Task 4: Web Push subscription endpoint + VAPID keys

**Files:**
- Modify: `apps/api/wrangler.toml` (add `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` as secrets)
- Create: `apps/api/src/services/push-vapid.ts`
- Create: `apps/api/src/routes/push.ts`
- Create: `apps/api/src/db/migrations/0003_push_subscriptions.sql` + `apps/api/src/db/schema/push_subscriptions.ts`
- Test: `apps/api/test/routes/push.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /push/subscribe stores endpoint+keys for current user, idempotent on same endpoint', async () => { /* ... */ });
it('DELETE /push/subscribe/:id removes subscription, only owner can delete', async () => { /* ... */ });
```

- [ ] **Step 2: Schema** `push_subscriptions(id, tenant_id, user_id, endpoint UNIQUE, p256dh, auth, ua, created_at)`.

- [ ] **Step 3: Implement** routes; generate VAPID keys with `bun run scripts/gen-vapid.ts`, store as wrangler secrets.

- [ ] **Step 4: Commit**

---

### Task 5: Web Push adapter + dispatcher integration

**Files:**
- Create: `apps/api/src/adapters/push-web.ts`
- Modify: `apps/api/src/services/notification-dispatcher.ts` (add `web_push` channel)
- Test: `apps/api/test/adapters/push.test.ts`

- [ ] **Step 1: Failing test** — push adapter sends to `endpoint` with VAPID-signed `Authorization: vapid t=..., k=...` header; on `410 Gone`, marks subscription deleted.

- [ ] **Step 2: Implement** using `web-push` polyfill compatible with Workers (uses WebCrypto for ECDSA P-256 sign).

- [ ] **Step 3: Add `web_push` to `notification_prefs_json` ordering**.

- [ ] **Step 4: Commit**

---

### Task 6: Client-side push subscribe flow

**Files:**
- Create: `apps/web/src/pwa/push-subscribe.ts`
- Create: `apps/web/src/pwa/sw-push-handler.ts` (custom SW message handler)
- Modify: `apps/web/src/features/notifications/NotificationCenter.tsx` (add "啟用推播" button)
- Test: `apps/web/e2e/push.spec.ts`

- [ ] **Step 1: Failing E2E (Playwright with `--enable-features=NotificationsBackend`)** — user grants permission, subscribes, server sees row in `push_subscriptions`; trigger alert in DB → push received → SW shows notification → click → opens order detail.

- [ ] **Step 2: Implement** subscribe via `pushManager.subscribe({ userVisibleOnly:true, applicationServerKey })`; SW `push` event handler; SW `notificationclick` opens `event.notification.data.url`.

- [ ] **Step 3: Commit**

---

### Task 7: Barcode detector hook

**Files:**
- Create: `apps/web/src/features/scan/useBarcodeDetector.ts`
- Test: `apps/web/test/scan/detector.test.ts`

- [ ] **Step 1: Failing test (mock `BarcodeDetector`)** — hook returns `{ supported, scan, lastResult, error }`; when `BarcodeDetector` is undefined, `supported=false`; when call detects code → `lastResult.rawValue` updates.

- [ ] **Step 2: Implement**
```ts
export function useBarcodeDetector(formats: string[] = ['code_128','ean_13','qr_code']) {
  const supported = 'BarcodeDetector' in window;
  const detectorRef = useRef<any>();
  useEffect(() => {
    if (!supported) return;
    detectorRef.current = new (window as any).BarcodeDetector({ formats });
  }, []);
  const scan = useCallback(async (videoEl: HTMLVideoElement) => {
    if (!detectorRef.current) return null;
    const codes = await detectorRef.current.detect(videoEl);
    return codes[0]?.rawValue ?? null;
  }, []);
  return { supported, scan };
}
```

- [ ] **Step 3: Commit**

> Per spec §10: no ZXing fallback. If `supported=false`, show "您的瀏覽器不支援掃碼，請改用 Android Chrome 或 iOS 16+ Safari".

---

### Task 8: BarcodeScanner component + camera permission flow

**Files:**
- Create: `apps/web/src/features/scan/BarcodeScanner.tsx`, `ScanFeedback.tsx`
- Create: `apps/web/src/routes/scan.tsx`
- Test: `apps/web/e2e/scan.spec.ts`

- [ ] **Step 1: Failing E2E (with camera mock)**
```ts
test('scan view requests camera, decodes mock barcode, beep+vibrate, shows decoded value', async ({ page, context }) => {
  await context.grantPermissions(['camera']);
  await page.goto('/scan');
  // playwright: replace getUserMedia with fake video stream containing test barcode
  await expect(page.getByTestId('scan-result')).toContainText('SKU-1234');
});
```

- [ ] **Step 2: Implement** `<video autoPlay playsInline muted>` + `requestAnimationFrame` loop calling `scan()` at 4 FPS; haptic via `navigator.vibrate(50)` + WebAudio beep; throttle duplicate codes within 1.5s.

- [ ] **Step 3: Commit**

---

### Task 9: Picking API (scan → mark line picked)

**Files:**
- Create: `apps/api/src/routes/picking.ts`
- Modify: `apps/api/src/db/migrations/0004_picking.sql` + `order_items` add `picked_qty`, `picked_at`
- Test: `apps/api/test/routes/picking.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /orders/:id/picking/scan with sku increments picked_qty by 1, blocks > qty', async () => { /* ... */ });
it('returns picking_complete:true when sum(picked_qty) === sum(qty)', async () => { /* ... */ });
it('only ops|admin can pick, sales gets 403', async () => { /* ... */ });
it('current stage must be "picking" else 409', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** atomic update inside transaction.

- [ ] **Step 3: Commit**

---

### Task 10: PickingView UI

**Files:**
- Create: `apps/web/src/features/picking/PickingView.tsx`, `PickingItemRow.tsx`, `PickingComplete.tsx`
- Create: `apps/web/src/routes/picking/$orderId.tsx`
- Test: `apps/web/e2e/picking.spec.ts`

- [ ] **Step 1: Failing E2E** — ops opens `/picking/<orderId>`, scans barcodes (mocked) for each item, sees count increment, on 100% sees "完成揀貨 → 推進到包裝" button → click → advances stage.

- [ ] **Step 2: Implement** — list of items with progress bars; scanner overlay; on `picking_complete`, button calls `/stages/advance` directly.

- [ ] **Step 3: Mobile-first layout** with one-thumb operation, big tap targets (min 56px).

- [ ] **Step 4: Commit**

---

### Task 11: Manual quantity adjustment + barcode error recovery

**Files:**
- Modify: `PickingView.tsx`, `picking.ts` route (add `?action=set&qty=`)
- Test: `picking.test.ts` (add cases)

- [ ] **Step 1: Failing test** — ops over-scans by mistake → can manually set `picked_qty=N`; sets `picked_qty > qty` rejected (400).

- [ ] **Step 2: Implement** edit dialog on row tap.

- [ ] **Step 3: Commit**

---

### Task 12: Mobile-friendly order list

**Files:**
- Modify: `apps/web/src/routes/orders/index.tsx`, `list.tsx`
- Create: `apps/web/src/features/orders/OrderListMobile.tsx`

- [ ] **Step 1: Failing test (visual)** — at viewport 375x667, kanban switches to single-column scroll view with collapsed stage filter.

- [ ] **Step 2: Implement** — useMediaQuery, swap component on mobile; large tap targets; sticky filter bar.

- [ ] **Step 3: Commit**

---

### Task 13: Team-scoped queries (sales_manager)

**Files:**
- Modify: `apps/api/src/middleware/rbac.ts` (add `scopeOrders` helper that injects `team_id IN (user.managed_team_ids)` for sales_manager)
- Modify: `apps/api/src/routes/orders.ts`
- Test: `apps/api/test/services/team-scope.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('sales_manager sees orders from members of their team(s) only', async () => {
  const seedA = await seed.salesUser({ teamId:'T1' });
  const seedB = await seed.salesUser({ teamId:'T2' });
  const mgr = await seed.salesManager({ managedTeamIds:['T1'] });
  const oA = await seed.order({ salesUserId: seedA.id });
  const oB = await seed.order({ salesUserId: seedB.id });
  const res = await app.request('/api/orders', { headers: authHeaders(mgr.id, mgr.tenantId) });
  const ids = (await res.json()).items.map(o=>o.id);
  expect(ids).toContain(oA.id);
  expect(ids).not.toContain(oB.id);
});
it('sales user sees only own orders', async () => { /* ... */ });
it('admin sees all tenant orders', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** — `users.managed_team_ids` is JSON array; build SQL `WHERE` per role.

- [ ] **Step 3: Commit**

---

### Task 14: Sales manager dashboard

**Files:**
- Create: `apps/web/src/routes/dashboards/sales.tsx` (manager only)
- Test: `apps/web/e2e/manager-dashboard.spec.ts`

- [ ] **Step 1: Failing E2E** — manager logs in, sees team pipeline (orders by stage), drill-down to per-salesperson breakdown.

- [ ] **Step 2: Implement** with TanStack Query + recharts (already in Phase 1).

- [ ] **Step 3: Commit**

---

### Task 15: Payments (deposit + final) — API

**Files:**
- Create: `apps/api/src/routes/payments.ts`
- Modify: `apps/api/src/db/migrations/0005_payments.sql` — table `payments(id, tenant_id, order_id, kind, amount, paid_at, ref_no, recorded_by, created_at)` where kind ∈ `{deposit, final}`
- Test: `apps/api/test/routes/payments.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('POST /orders/:id/payments creates payment, advances stage if matching (deposit→deposit_paid, final→final_paid)', async () => { /* ... */ });
it('only accountant|admin can record payment; sales 403', async () => { /* ... */ });
it('total deposit + final cannot exceed order total_amount', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** — atomic insert + stage advance + audit.

- [ ] **Step 3: Commit**

---

### Task 16: Payments UI (accountant)

**Files:**
- Create: `apps/web/src/features/payments/PaymentDialog.tsx`, `PaymentList.tsx`
- Create: `apps/web/src/routes/payments/index.tsx` (accountant default home)
- Modify: `OrderDetail` (show payments section to accountant only)

- [ ] **Step 1: Failing E2E** — accountant opens orders awaiting deposit, marks paid, stage advances; sales user does not see payment record/edit buttons.

- [ ] **Step 2: Implement**

- [ ] **Step 3: Commit**

---

### Task 17: High-value approval queue UI (admin/sales_manager)

**Files:**
- Create: `apps/api/src/routes/approvals.ts` (GET pending, POST approve/reject)
- Create: `apps/web/src/features/approvals/ApprovalQueue.tsx`
- Create: `apps/web/src/routes/approvals/index.tsx`
- Test: `apps/web/e2e/approvals.spec.ts`

- [ ] **Step 1: Failing E2E** — manager sees high-value orders pending approval, opens detail, approves with note → order proceeds, sales gets notification; reject sets `status='rejected'`, alerts sales.

- [ ] **Step 2: Implement** — wire to Phase 1 `POST /orders/:id/approve` + add `/reject` route.

- [ ] **Step 3: Commit**

---

### Task 18: AR aging report

**Files:**
- Create: `apps/api/src/services/aging-calculator.ts`
- Modify: `apps/api/src/routes/reports.ts` (add `/reports/aging`)
- Create: `apps/web/src/features/dashboards/AgingReport.tsx`
- Create: `apps/web/src/routes/dashboards/aging.tsx`
- Test: `apps/api/test/services/aging.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('aging buckets unpaid balances by days since invoice_date: 0-30, 31-60, 61-90, 90+', async () => {
  const result = await calcAging(db, 'tenantA', new Date('2026-04-30'));
  expect(result.buckets['0-30']).toBe(150000);
  expect(result.buckets['90+']).toBe(50000);
});
```

- [ ] **Step 2: Implement** — `unpaid = total_amount - sum(payments.amount)`; bucket by `now - invoice.created_at` (use latest invoice doc's created_at as proxy until proper invoice_date column added).

- [ ] **Step 3: UI** stacked bar + table, export CSV.

- [ ] **Step 4: Role gate**: accountant + admin only.

- [ ] **Step 5: Commit**

---

### Task 19: SLA dashboard

**Files:**
- Create: `apps/web/src/features/dashboards/SlaDashboard.tsx`
- Create: `apps/web/src/routes/dashboards/sla.tsx`
- Modify: `apps/api/src/routes/reports.ts` (add `/reports/sla/live`)
- Test: `apps/web/e2e/sla-dashboard.spec.ts`

- [ ] **Step 1: Failing E2E** — admin sees: orders at risk (within 24h of SLA), already breached, breaches by stage chart; click order → detail.

- [ ] **Step 2: Implement** SQL for `at_risk = started_at + sla_hours - now < 24h` and `breached = started_at + sla_hours < now`.

- [ ] **Step 3: Auto-refresh every 60s.**

- [ ] **Step 4: Commit**

---

### Task 20: Money field redaction by role

**Files:**
- Create: `apps/api/src/middleware/redact-money.ts`
- Modify: `apps/api/src/routes/orders.ts` + `customers.ts` (apply redactor to responses)
- Test: `apps/api/test/middleware/redact.test.ts`

- [ ] **Step 1: Failing test**
```ts
it('viewer role: total_amount, unit_price, payments are redacted to null', async () => { /* ... */ });
it('accountant: full money fields visible', async () => { /* ... */ });
it('ops: total_amount visible (production planning) but unit_price hidden', async () => { /* ... */ });
```

- [ ] **Step 2: Implement** declarative field map per role; apply via Hono response middleware.

- [ ] **Step 3: Commit**

---

### Task 21: Approval flow integrated with notifications

**Files:**
- Modify: `apps/api/src/services/alert-evaluator.ts` — when order created with `requires_approval=true`, immediately enqueue notification to managers + admin
- Modify: `apps/api/src/routes/orders.ts` (POST `/approve`, `/reject` notify back to sales)
- Test: `apps/api/test/flows/approval.test.ts`

- [ ] **Step 1: Failing test** end-to-end: sales creates 600k order → manager + admin get email/teams → manager approves → sales gets notification → order continues.

- [ ] **Step 2: Implement**

- [ ] **Step 3: Commit**

---

### Task 22: Phase 2 verification + smoke E2E

**Files:**
- Create: `apps/web/e2e/phase2-smoke.spec.ts`
- Modify: `README.md`

- [ ] **Step 1: Smoke E2E**
```ts
test('phase 2 happy path: install PWA → enable push → ops scans 3 SKUs → packing complete → admin approves high-value → accountant marks final paid → sla dashboard reflects', async ({ page, context }) => { /* ... */ });
```

- [ ] **Step 2: Manual checklist**
- [ ] iOS Safari 16+: install to home screen, app launches standalone, push notifications work
- [ ] Android Chrome: install banner appears, BarcodeDetector decodes Code128 + EAN13
- [ ] Web push delivered within 5s of alert
- [ ] sales_manager sees only their team's orders
- [ ] accountant payment recorded → stage advances atomically
- [ ] viewer cannot see total_amount anywhere
- [ ] high-value order blocks ship until approved
- [ ] AR aging numbers reconcile against `total - paid` per order

- [ ] **Step 3: Tag**
```bash
git tag -a v0.2.0-phase2 -m "Phase 2 mobile + advanced perms complete"
```

- [ ] **Step 4: Final commit**

---

## Phase 2 Done When

1. All 22 tasks ✅
2. PWA passes Lighthouse PWA audit ≥ 90
3. Smoke E2E passes
4. Manual checklist all confirmed
5. No SW caches sensitive API responses (audit `runtimeCaching` config; only short TTL, no auth-bearing GETs cached >60s)
6. Phase 3 can begin (MCP + multi-tenant signup)
