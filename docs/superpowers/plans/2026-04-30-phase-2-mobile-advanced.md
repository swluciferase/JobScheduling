# OrderFlow Phase 2: Mobile PWA + UDI Barcode + Advanced Permissions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make OrderFlow usable on the factory floor (PWA + UDI/Lot barcode scanning at incoming inspection, in-process QC, final QC, and shipment) and complete the advanced permission features required for QMS operations (sales_manager team scope, accountant money fields, qa_manager quality dashboards, periodic access review per ISO 27001 A.9.2.5).

**Architecture:** Layer on top of Phase 1 MVP. Convert the web app into an installable PWA with a service worker (cache-first for shell, network-first for API — **no offline writes** to preserve ALCOA+ contemporaneous attribute). Add `BarcodeDetector`-based UDI/Lot scanner that parses GS1 Application Identifiers (AI 01 GTIN/DI, AI 10 Lot/PI, AI 17 Expiry, AI 21 Serial). Wire scanner into incoming inspection (verify supplier UDI), in-process QC (verify Lot in WIP), final QC (verify Lot in finished goods), and packed_shipped (verify Serial/UDI matches DHR before shipment). Introduce qa_manager dashboards (NCR/CAPA/audit overdue counts), team-scoped sales queries, accountant payment views, AR aging, and a quarterly access review workflow with admin e-signature.

**Tech Stack:** Same as Phase 1 + `vite-plugin-pwa` (manifest + Workbox SW), Web Push API + VAPID, BarcodeDetector (native, no fallback per spec §10), `gs1-barcode-parser-mod` (or hand-rolled GS1 AI parser — small footprint preferred), recharts (charts already in Phase 1), date-fns for aging buckets.

**Phase boundary:** Ends when (a) an ops user can install the app on iOS Safari ≥16 / Android Chrome, scan a UDI/Lot barcode at incoming inspection / QC / shipment stages with the scan recorded into the acceptance record and audit log; (b) a sales_manager can view their team's pipeline; (c) an accountant can mark deposit/final paid and run AR aging; (d) a qa_manager can view a quality dashboard (open NCR/CAPA, overdue audits); (e) an admin can run a quarterly access review with e-signed sign-off; (f) the SLA dashboard shows live stage breach risk per QMS template. NCR/CAPA/Complaints/Document Control/Training Records deferred to Phase 3.

---

## Reference Spec

Cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §3 Regulatory mapping (UDI Rule 21 CFR 830, ISO 27001 A.9.2.5 access review)
- §4 RBAC matrix (sales_manager team scope, accountant money fields, qa_manager quality scope, viewer read-only)
- §5 Lot/UDI/Serial traceability (GS1 AI parsing requirement)
- §6 Stage gating with e-sig at incoming_inspection, in_process_qc, final_qc, release_for_shipment, packed_shipped
- §10 Mobile / barcode scanning (no offline writes; BarcodeDetector only; iOS Safari 16+)
- §12 Reports (AR aging, SLA dashboard, qa_manager quality dashboard)
- §16 Access management (quarterly review, role/team change e-sig, account disable on offboard)

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
│   │   │   │   ├── BarcodeScanner.tsx        # Camera + BarcodeDetector
│   │   │   │   ├── ScanFeedback.tsx          # Pass/Fail/Mismatch UI
│   │   │   │   ├── useBarcodeDetector.ts     # Hook around native API
│   │   │   │   └── gs1-parser.ts             # GS1 AI parser
│   │   │   ├── inspection/
│   │   │   │   ├── IncomingInspectionView.tsx  # Scan supplier UDI/Lot
│   │   │   │   ├── InProcessQcView.tsx
│   │   │   │   ├── FinalQcView.tsx
│   │   │   │   └── ShipmentVerifyView.tsx    # Scan before pack/ship
│   │   │   ├── approvals/
│   │   │   │   └── ApprovalQueue.tsx
│   │   │   ├── accounting/
│   │   │   │   ├── PaymentMarkView.tsx
│   │   │   │   └── ArAgingReport.tsx
│   │   │   ├── qa-dashboard/
│   │   │   │   ├── QaOverviewView.tsx
│   │   │   │   └── widgets/
│   │   │   │       ├── OpenNcrCard.tsx
│   │   │   │       ├── OpenCapaCard.tsx
│   │   │   │       ├── OverdueAuditsCard.tsx
│   │   │   │       └── StageBreachCard.tsx
│   │   │   ├── access-review/
│   │   │   │   ├── AccessReviewView.tsx       # Quarterly review UI
│   │   │   │   ├── ReviewRoleAssignmentDialog.tsx
│   │   │   │   └── ReviewSignOffDialog.tsx    # E-sig sign-off
│   │   │   ├── teams/
│   │   │   │   ├── TeamPipelineView.tsx       # Sales manager view
│   │   │   │   └── TeamMembersTable.tsx
│   │   │   └── sla-dashboard/
│   │   │       ├── SlaDashboardView.tsx
│   │   │       └── BreachRiskTable.tsx
├── apps/api/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── scan.ts                        # POST /api/scan/verify (UDI lookup)
│   │   │   ├── inspection.ts                  # Tied to acceptance records
│   │   │   ├── teams.ts                       # GET /api/teams, /me/team
│   │   │   ├── ar-aging.ts
│   │   │   ├── qa-dashboard.ts
│   │   │   ├── access-review.ts               # quarterly review CRUD
│   │   │   └── sla-dashboard.ts
│   │   ├── services/
│   │   │   ├── teamScope.ts                   # Apply team filter to queries
│   │   │   ├── accessReviewService.ts
│   │   │   ├── arAgingService.ts
│   │   │   └── slaBreachService.ts
│   │   └── db/
│   │       └── migrations/
│   │           └── 0003_phase2_perms.sql       # teams, access_reviews, push_subscriptions
└── apps/web/
    └── vite.config.ts                          # Add VitePWA plugin
```

---

## Task List

### Week 7-8: PWA shell + UDI/Lot barcode foundation

#### Task 1: Add VitePWA plugin and manifest

**Files:**
- Modify: `apps/web/vite.config.ts`
- Create: `apps/web/public/manifest.webmanifest`
- Create: `apps/web/public/icon-192.png`, `icon-512.png`, `apple-touch-icon.png` (placeholders OK)

- [ ] **Step 1: Install vite-plugin-pwa**

```bash
cd apps/web && bun add -D vite-plugin-pwa workbox-window
```

- [ ] **Step 2: Configure VitePWA in vite.config.ts**

```ts
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'prompt',
      injectRegister: false,
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        navigateFallbackDenylist: [/^\/api/],
        runtimeCaching: [
          { urlPattern: /\/api\//, handler: 'NetworkOnly' },
          { urlPattern: /\.(png|svg|woff2)$/, handler: 'CacheFirst', options: { cacheName: 'static-assets' } }
        ]
      },
      manifest: false
    })
  ]
});
```

> **Why NetworkOnly for /api:** ALCOA+ "contemporaneous" attribute forbids offline writes. All e-sig and stage advances must hit live API.

- [ ] **Step 3: Write manifest.webmanifest**

```json
{
  "name": "OrderFlow QMS",
  "short_name": "OrderFlow",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#1e40af",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

- [ ] **Step 4: Build and verify SW registers**

Run: `bun run build && bun run preview`
Expected: DevTools → Application → Service Workers shows `sw.js` registered.

- [ ] **Step 5: Commit**

```bash
git add apps/web/vite.config.ts apps/web/public/manifest.webmanifest apps/web/public/icon-*.png apps/web/package.json
git commit -m "feat(phase2): add PWA shell — VitePWA + manifest, NetworkOnly for /api"
```

---

#### Task 2: SW register + install prompt component

**Files:**
- Create: `apps/web/src/pwa/register-sw.ts`
- Create: `apps/web/src/pwa/install-prompt.tsx`
- Modify: `apps/web/src/main.tsx`

- [ ] **Step 1: Write the failing test**

```ts
// apps/web/src/pwa/__tests__/install-prompt.test.tsx
import { render, screen } from '@testing-library/react';
import { InstallPrompt } from '../install-prompt';

test('shows install button when beforeinstallprompt fires', () => {
  render(<InstallPrompt />);
  window.dispatchEvent(new Event('beforeinstallprompt'));
  expect(screen.getByRole('button', { name: /install/i })).toBeInTheDocument();
});
```

- [ ] **Step 2: Run test (expect fail)**

Run: `bun test apps/web/src/pwa/__tests__/install-prompt.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implement register-sw.ts**

```ts
import { registerSW } from 'virtual:pwa-register';

export function registerServiceWorker() {
  if ('serviceWorker' in navigator) {
    registerSW({ immediate: true });
  }
}
```

- [ ] **Step 4: Implement install-prompt.tsx**

```tsx
import { useEffect, useState } from 'react';
import { Button } from '@/components/ui/button';

export function InstallPrompt() {
  const [evt, setEvt] = useState<any>(null);
  useEffect(() => {
    const h = (e: any) => { e.preventDefault(); setEvt(e); };
    window.addEventListener('beforeinstallprompt', h);
    return () => window.removeEventListener('beforeinstallprompt', h);
  }, []);
  if (!evt) return null;
  return <Button onClick={() => evt.prompt()}>Install OrderFlow</Button>;
}
```

- [ ] **Step 5: Wire into main.tsx**

```tsx
registerServiceWorker();
```

- [ ] **Step 6: Verify test passes**

Run: `bun test apps/web/src/pwa/__tests__/install-prompt.test.tsx`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add apps/web/src/pwa/ apps/web/src/main.tsx
git commit -m "feat(phase2): SW register + install prompt component"
```

---

#### Task 3: GS1 AI barcode parser (UDI Rule 21 CFR 830)

**Files:**
- Create: `apps/web/src/features/scan/gs1-parser.ts`
- Create: `apps/web/src/features/scan/__tests__/gs1-parser.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
import { parseGs1 } from '../gs1-parser';

test('parses GTIN + lot + expiry', () => {
  const raw = '0100614141999996101ABC1234\u001D17260301';
  expect(parseGs1(raw)).toEqual({
    gtin: '00614141999996',  // AI 01 = DI
    lot: 'ABC1234',           // AI 10 = PI lot
    expiry: '2026-03-01'      // AI 17
  });
});

test('parses serial number for unique device', () => {
  const raw = '010061414199999621SN-0001';
  expect(parseGs1(raw)).toEqual({
    gtin: '00614141999996',
    serial: 'SN-0001'  // AI 21
  });
});

test('rejects malformed GS1', () => {
  expect(() => parseGs1('garbage')).toThrow(/invalid GS1/);
});
```

- [ ] **Step 2: Run tests (expect fail)**

Run: `bun test apps/web/src/features/scan/__tests__/gs1-parser.test.ts`
Expected: 3 FAIL.

- [ ] **Step 3: Implement parser**

```ts
const FIXED_LEN_AI: Record<string, number> = {
  '01': 14,  // GTIN
  '17': 6,   // Expiry YYMMDD
  '11': 6,   // Production date
};
const VAR_LEN_AI = new Set(['10', '21']);  // Lot, Serial — terminated by FNC1 (\u001D) or end
const GS = '\u001D';

export interface Gs1Parsed {
  gtin?: string;
  lot?: string;
  serial?: string;
  expiry?: string;        // ISO YYYY-MM-DD
  productionDate?: string;
}

export function parseGs1(raw: string): Gs1Parsed {
  const out: Gs1Parsed = {};
  let i = 0;
  while (i < raw.length) {
    const ai = raw.slice(i, i + 2);
    i += 2;
    if (FIXED_LEN_AI[ai]) {
      const v = raw.slice(i, i + FIXED_LEN_AI[ai]);
      i += FIXED_LEN_AI[ai];
      if (ai === '01') out.gtin = v;
      else if (ai === '17') out.expiry = `20${v.slice(0,2)}-${v.slice(2,4)}-${v.slice(4,6)}`;
      else if (ai === '11') out.productionDate = `20${v.slice(0,2)}-${v.slice(2,4)}-${v.slice(4,6)}`;
    } else if (VAR_LEN_AI.has(ai)) {
      const end = raw.indexOf(GS, i);
      const v = end === -1 ? raw.slice(i) : raw.slice(i, end);
      i = end === -1 ? raw.length : end + 1;
      if (ai === '10') out.lot = v;
      else if (ai === '21') out.serial = v;
    } else {
      throw new Error(`invalid GS1: unknown AI ${ai}`);
    }
  }
  if (!out.gtin) throw new Error('invalid GS1: missing GTIN');
  return out;
}
```

- [ ] **Step 4: Verify tests pass**

Run: `bun test apps/web/src/features/scan/__tests__/gs1-parser.test.ts`
Expected: 3 PASS.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/features/scan/gs1-parser.ts apps/web/src/features/scan/__tests__/gs1-parser.test.ts
git commit -m "feat(phase2): GS1 AI barcode parser (GTIN/Lot/Serial/Expiry per UDI Rule)"
```

---

#### Task 4: BarcodeDetector hook + scanner component

**Files:**
- Create: `apps/web/src/features/scan/useBarcodeDetector.ts`
- Create: `apps/web/src/features/scan/BarcodeScanner.tsx`
- Create: `apps/web/src/features/scan/ScanFeedback.tsx`

- [ ] **Step 1: Implement useBarcodeDetector hook**

```ts
import { useEffect, useRef, useState } from 'react';

export function useBarcodeDetector(formats: string[] = ['data_matrix','qr_code','code_128']) {
  const [detector, setDetector] = useState<any>(null);
  const [supported, setSupported] = useState(false);
  useEffect(() => {
    if ('BarcodeDetector' in window) {
      setSupported(true);
      setDetector(new (window as any).BarcodeDetector({ formats }));
    }
  }, []);
  return { detector, supported };
}
```

- [ ] **Step 2: Implement BarcodeScanner component**

```tsx
import { useEffect, useRef } from 'react';
import { useBarcodeDetector } from './useBarcodeDetector';

export interface BarcodeScannerProps {
  onDetected: (raw: string) => void;
  onError: (e: Error) => void;
}

export function BarcodeScanner({ onDetected, onError }: BarcodeScannerProps) {
  const videoRef = useRef<HTMLVideoElement>(null);
  const { detector, supported } = useBarcodeDetector();

  useEffect(() => {
    if (!supported || !detector) return;
    let stream: MediaStream | null = null;
    let raf = 0;
    let stopped = false;

    (async () => {
      try {
        stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
        if (videoRef.current) {
          videoRef.current.srcObject = stream;
          await videoRef.current.play();
        }
        const tick = async () => {
          if (stopped || !videoRef.current) return;
          try {
            const codes = await detector.detect(videoRef.current);
            if (codes.length > 0) onDetected(codes[0].rawValue);
          } catch {}
          raf = requestAnimationFrame(tick);
        };
        raf = requestAnimationFrame(tick);
      } catch (e) {
        onError(e as Error);
      }
    })();

    return () => {
      stopped = true;
      if (raf) cancelAnimationFrame(raf);
      stream?.getTracks().forEach(t => t.stop());
    };
  }, [detector, supported, onDetected, onError]);

  if (!supported) {
    return <div className="p-4 bg-red-50 text-red-700">此瀏覽器不支援掃碼。請使用 iOS Safari 16+ 或 Android Chrome。</div>;
  }

  return <video ref={videoRef} className="w-full rounded" autoPlay playsInline muted />;
}
```

- [ ] **Step 3: Implement ScanFeedback component (audio + visual)**

```tsx
export function ScanFeedback({ status }: { status: 'idle'|'pass'|'fail'|'mismatch' }) {
  const colors = { idle: 'bg-slate-100', pass: 'bg-emerald-500', fail: 'bg-red-500', mismatch: 'bg-amber-500' };
  return (
    <div className={`h-3 rounded transition-colors ${colors[status]}`} role="status" aria-label={status} />
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/scan/
git commit -m "feat(phase2): BarcodeScanner with native BarcodeDetector + visual feedback"
```

---

#### Task 5: Scan verify API endpoint (UDI/Lot lookup)

**Files:**
- Create: `apps/api/src/routes/scan.ts`
- Modify: `apps/api/src/index.ts`
- Create: `apps/api/src/routes/__tests__/scan.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
test('POST /api/scan/verify resolves order_item by gtin+lot', async () => {
  await seedOrderItem({ gtin: '00614141999996', lot: 'ABC1234', orderId: 'ord_1' });
  const res = await app.fetch(authedReq('/api/scan/verify', {
    method: 'POST', body: { gtin: '00614141999996', lot: 'ABC1234' }
  }));
  expect(res.status).toBe(200);
  const body = await res.json();
  expect(body.matches).toHaveLength(1);
  expect(body.matches[0].orderId).toBe('ord_1');
});

test('POST /api/scan/verify returns 200 with empty matches when not found', async () => {
  const res = await app.fetch(authedReq('/api/scan/verify', {
    method: 'POST', body: { gtin: '99999999999999', lot: 'NOTFOUND' }
  }));
  const body = await res.json();
  expect(body.matches).toEqual([]);
});
```

- [ ] **Step 2: Implement /api/scan/verify**

```ts
import { Hono } from 'hono';
import { z } from 'zod';
import { db } from '../db';
import { orderItems, orders } from '../db/schema';
import { eq, and } from 'drizzle-orm';

const ScanReq = z.object({
  gtin: z.string().min(1),
  lot: z.string().optional(),
  serial: z.string().optional(),
});

export const scanRoute = new Hono().post('/verify', async (c) => {
  const body = ScanReq.parse(await c.req.json());
  const conditions = [eq(orderItems.gtin, body.gtin)];
  if (body.lot) conditions.push(eq(orderItems.lot, body.lot));
  if (body.serial) conditions.push(eq(orderItems.serial, body.serial));
  const matches = await db.select().from(orderItems)
    .innerJoin(orders, eq(orders.id, orderItems.orderId))
    .where(and(...conditions))
    .limit(20);
  await writeAudit(db, {
    actor_user_id: c.get('user').id,
    action: 'scan.verify',
    entity_type: 'order_item',
    entity_id: matches[0]?.order_items.id ?? 'none',
    detail_json: JSON.stringify({ gtin: body.gtin, lot: body.lot, serial: body.serial, hitCount: matches.length })
  });
  return c.json({ matches: matches.map(r => ({ orderId: r.order_items.orderId, itemId: r.order_items.id, sku: r.order_items.sku, lot: r.order_items.lot, serial: r.order_items.serial, expiry: r.order_items.expiry, currentStage: r.orders.currentStageKey })) });
});
```

- [ ] **Step 3: Verify tests pass**

Run: `bun test apps/api/src/routes/__tests__/scan.test.ts`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/routes/scan.ts apps/api/src/index.ts apps/api/src/routes/__tests__/scan.test.ts
git commit -m "feat(phase2): /api/scan/verify — UDI/Lot lookup with audit log"
```

---

### Week 9: Inspection scan flows + acceptance record integration

#### Task 6: Incoming inspection scan view (820.80(a))

**Files:**
- Create: `apps/web/src/features/inspection/IncomingInspectionView.tsx`
- Modify: `apps/web/src/features/orders/OrderDetailPage.tsx` (add CTA when stage = `incoming_inspection` and user role ∈ {ops, qa_manager})

- [ ] **Step 1: Implement IncomingInspectionView**

```tsx
export function IncomingInspectionView({ orderId, itemId }: { orderId: string; itemId: string }) {
  const [scanned, setScanned] = useState<Gs1Parsed | null>(null);
  const [verifying, setVerifying] = useState(false);
  const [status, setStatus] = useState<'idle'|'pass'|'fail'|'mismatch'>('idle');
  const item = useOrderItem(orderId, itemId);

  const onDetected = async (raw: string) => {
    let parsed: Gs1Parsed;
    try { parsed = parseGs1(raw); } catch { setStatus('fail'); return; }
    setScanned(parsed);
    setVerifying(true);
    // Verify against expected supplier UDI on PO line
    const matches = parsed.gtin === item.expectedGtin && (!item.expectedLot || parsed.lot === item.expectedLot);
    setStatus(matches ? 'pass' : 'mismatch');
    setVerifying(false);
  };

  const recordAcceptance = async (result: 'pass'|'fail') => {
    await api.post(`/api/orders/${orderId}/items/${itemId}/inspection`, {
      stage: 'incoming_inspection',
      gtin: scanned?.gtin,
      lot: scanned?.lot,
      expiry: scanned?.expiry,
      result,
      method: 'visual+barcode-scan',
    });
  };

  return (
    <div className="space-y-4">
      <BarcodeScanner onDetected={onDetected} onError={() => setStatus('fail')} />
      <ScanFeedback status={status} />
      {scanned && (
        <Card>
          <CardContent>
            <div>GTIN: {scanned.gtin}</div>
            <div>Lot: {scanned.lot ?? '—'}</div>
            <div>Expiry: {scanned.expiry ?? '—'}</div>
            <div className="flex gap-2 mt-4">
              <Button onClick={() => recordAcceptance('pass')} disabled={status !== 'pass'}>Accept</Button>
              <Button variant="destructive" onClick={() => recordAcceptance('fail')}>Reject → NCR</Button>
            </div>
          </CardContent>
        </Card>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Wire into order detail page conditional render**

- [ ] **Step 3: Manual smoke test**

Open order in `incoming_inspection` stage → click "Inspection Scan" → camera opens → scan a sample GS1 barcode → verify pass/mismatch UI → click Accept → acceptance record created.

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/inspection/IncomingInspectionView.tsx apps/web/src/features/orders/
git commit -m "feat(phase2): incoming inspection scan view with UDI verification"
```

---

#### Task 7: In-process QC + Final QC scan views

**Files:**
- Create: `apps/web/src/features/inspection/InProcessQcView.tsx`
- Create: `apps/web/src/features/inspection/FinalQcView.tsx`

- [ ] **Step 1: Implement both views (similar pattern to IncomingInspectionView)**

Differences:
- In-process: scan WIP Lot, verify against work order Lot, record `in_process_qc` acceptance
- Final QC: scan finished-goods Lot/Serial, verify against order item Lot/Serial, record `final_qc` acceptance with sampling plan reference (free-form text field)

- [ ] **Step 2: Both views must trigger e-sig dialog before recording acceptance** (per spec §6 — `in_process_qc` and `final_qc` are e-sig stages)

Reuse `<ESignDialog>` from Phase 1 Task 19.

- [ ] **Step 3: Manual smoke test for both flows**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/inspection/InProcessQcView.tsx apps/web/src/features/inspection/FinalQcView.tsx
git commit -m "feat(phase2): in-process QC + final QC scan views with e-sig"
```

---

#### Task 8: Shipment verification scan view (pre-pack/ship)

**Files:**
- Create: `apps/web/src/features/inspection/ShipmentVerifyView.tsx`

- [ ] **Step 1: Implement view**

Flow:
1. Display all line items in order with their expected GTIN/Lot/Serial.
2. Operator scans each unit; system marks line item "verified" when GTIN+Lot+Serial all match DHR draft.
3. When all items verified, "Pack & Ship" button enabled → triggers e-sig dialog → advances stage to `packed_shipped`.
4. Mismatched scan blocks advance and offers "Open NCR" CTA (NCR module is Phase 3 stub-OK).

- [ ] **Step 2: Verify all line items must be scanned (no partial advance)**

- [ ] **Step 3: Audit log entry: `shipment.verify_scan` per item with full GTIN/Lot/Serial detail**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/inspection/ShipmentVerifyView.tsx
git commit -m "feat(phase2): shipment verify view — all units scanned before pack/ship"
```

---

### Week 10: Advanced permissions — teams + accountant + qa_manager

#### Task 9: Phase 2 DB migration (teams + push subscriptions + access_reviews)

**Files:**
- Create: `apps/api/src/db/migrations/0003_phase2_perms.sql`
- Modify: `apps/api/src/db/schema.ts`

- [ ] **Step 1: Write migration SQL**

```sql
-- Sales teams (sales_manager scope)
CREATE TABLE teams (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  manager_user_id TEXT NOT NULL REFERENCES users(id),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

ALTER TABLE users ADD COLUMN team_id TEXT REFERENCES teams(id);
CREATE INDEX idx_users_team ON users(team_id);

ALTER TABLE orders ADD COLUMN team_id TEXT REFERENCES teams(id);
CREATE INDEX idx_orders_team ON orders(team_id);

-- Web Push subscriptions (per user, multiple devices)
CREATE TABLE push_subscriptions (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  endpoint TEXT NOT NULL,
  p256dh TEXT NOT NULL,
  auth TEXT NOT NULL,
  user_agent TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, endpoint)
);

-- Quarterly access reviews (ISO 27001 A.9.2.5 + 21 CFR 11.10(d))
CREATE TABLE access_reviews (
  id TEXT PRIMARY KEY,
  period TEXT NOT NULL,                         -- e.g. '2026-Q2'
  initiated_by TEXT NOT NULL REFERENCES users(id),
  initiated_at TIMESTAMP NOT NULL,
  completed_at TIMESTAMP,
  completed_by TEXT REFERENCES users(id),
  signoff_esig_id TEXT REFERENCES e_signatures(id),
  status TEXT NOT NULL DEFAULT 'in_progress'    -- in_progress | completed | cancelled
);

CREATE TABLE access_review_items (
  id TEXT PRIMARY KEY,
  review_id TEXT NOT NULL REFERENCES access_reviews(id),
  user_id TEXT NOT NULL REFERENCES users(id),
  prior_role TEXT NOT NULL,
  prior_team_id TEXT,
  decision TEXT NOT NULL,                        -- keep | change_role | change_team | disable
  new_role TEXT,
  new_team_id TEXT,
  reason TEXT NOT NULL,
  reviewed_at TIMESTAMP NOT NULL,
  reviewed_by TEXT NOT NULL REFERENCES users(id)
);
CREATE INDEX idx_review_items_review ON access_review_items(review_id);
```

- [ ] **Step 2: Update Drizzle schema**

- [ ] **Step 3: Apply migration to local D1**

Run: `bun run db:migrate:local`
Expected: tables created, FK constraints valid.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/db/migrations/0003_phase2_perms.sql apps/api/src/db/schema.ts
git commit -m "feat(phase2): db migration — teams + push subs + access_reviews"
```

---

#### Task 10: Team scope enforcement service

**Files:**
- Create: `apps/api/src/services/teamScope.ts`
- Create: `apps/api/src/services/__tests__/teamScope.test.ts`

- [ ] **Step 1: Write failing tests**

```ts
test('sales user sees only own orders', async () => {
  const filter = await applyOrderScope({ user: { id: 'u1', role: 'sales', teamId: 't1' } });
  expect(filter).toMatchObject({ ownerUserId: 'u1' });
});

test('sales_manager sees all orders in own team', async () => {
  const filter = await applyOrderScope({ user: { id: 'mgr1', role: 'sales_manager', teamId: 't1' } });
  expect(filter).toMatchObject({ teamId: 't1' });
});

test('admin/qa_manager sees all orders', async () => {
  for (const role of ['admin','qa_manager']) {
    const filter = await applyOrderScope({ user: { id: 'x', role, teamId: null } });
    expect(filter).toEqual({});
  }
});
```

- [ ] **Step 2: Implement service**

```ts
export function applyOrderScope({ user }: { user: AuthUser }): Record<string, any> {
  switch (user.role) {
    case 'admin':
    case 'qa_manager':
    case 'accountant':
    case 'viewer':
      return {};
    case 'sales_manager':
      if (!user.teamId) throw new Error('sales_manager must have team_id');
      return { teamId: user.teamId };
    case 'sales':
      return { ownerUserId: user.id };
    case 'ops':
      return {};   // ops sees all in production-related stages, but writes restricted by stage rules
    default:
      throw new Error(`unknown role ${user.role}`);
  }
}
```

- [ ] **Step 3: Apply scope in /api/orders list query**

Modify `routes/orders.ts` GET handler to call `applyOrderScope` and append filter.

- [ ] **Step 4: Verify tests pass + add integration test for /api/orders list**

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/services/teamScope.ts apps/api/src/services/__tests__/teamScope.test.ts apps/api/src/routes/orders.ts
git commit -m "feat(phase2): team scope service + apply to /api/orders list"
```

---

#### Task 11: Team pipeline view (sales_manager UI)

**Files:**
- Create: `apps/web/src/features/teams/TeamPipelineView.tsx`
- Create: `apps/web/src/features/teams/TeamMembersTable.tsx`
- Modify: `apps/web/src/routes.tsx` (add `/team` route, gated by role)

- [ ] **Step 1: Implement TeamPipelineView**

Show: pipeline by stage (received → … → packed_shipped) for own team's orders, total open orders, top customers, sales rep ranking.

- [ ] **Step 2: Implement TeamMembersTable** (read-only list of team members + active order counts)

- [ ] **Step 3: Add route gate**

```tsx
{user.role === 'sales_manager' && <Route path="/team" element={<TeamPipelineView />} />}
```

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/teams/ apps/web/src/routes.tsx
git commit -m "feat(phase2): team pipeline view for sales_manager"
```

---

#### Task 12: Accountant payment view

**Files:**
- Create: `apps/web/src/features/accounting/PaymentMarkView.tsx`
- Create: `apps/api/src/routes/payments.ts`

- [ ] **Step 1: Implement /api/orders/:id/payments POST endpoint**

Records `deposit_paid` or `final_paid` event with amount/method/reference. Triggers stage advance if matching stage active.

```ts
const PaymentReq = z.object({
  stage: z.enum(['deposit_paid','final_paid']),
  amount: z.number().positive(),
  method: z.string(),
  reference: z.string().optional(),
});
```

- [ ] **Step 2: Implement PaymentMarkView**

UI: list of orders with current stage = deposit_paid or final_paid pending; "Mark Paid" button → dialog → submit → audit log entry.

- [ ] **Step 3: Wire RBAC: only `accountant` and `admin` can write payments; sales/ops can read amounts but not mark paid.**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/accounting/PaymentMarkView.tsx apps/api/src/routes/payments.ts
git commit -m "feat(phase2): accountant payment marking view + API"
```

---

#### Task 13: AR aging report

**Files:**
- Create: `apps/api/src/services/arAgingService.ts`
- Create: `apps/api/src/routes/ar-aging.ts`
- Create: `apps/web/src/features/accounting/ArAgingReport.tsx`

- [ ] **Step 1: Aging buckets: 0-30, 31-60, 61-90, 91-180, 181+ days from invoice date**

- [ ] **Step 2: Implement service**

```ts
export async function computeArAging(asOf: Date) {
  const orders = await db.select().from(/* orders with final_paid not yet recorded */);
  return orders.reduce((acc, o) => {
    const daysOpen = daysBetween(o.invoiceDate, asOf);
    const bucket = bucketOf(daysOpen);
    acc[bucket] = (acc[bucket] ?? 0) + o.outstandingAmount;
    return acc;
  }, {});
}
```

- [ ] **Step 3: UI with chart (recharts) + drill-down table**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/arAgingService.ts apps/api/src/routes/ar-aging.ts apps/web/src/features/accounting/ArAgingReport.tsx
git commit -m "feat(phase2): AR aging report with bucket chart"
```

---

#### Task 14: QA manager dashboard (overview)

**Files:**
- Create: `apps/web/src/features/qa-dashboard/QaOverviewView.tsx`
- Create: `apps/web/src/features/qa-dashboard/widgets/`
- Create: `apps/api/src/routes/qa-dashboard.ts`

- [ ] **Step 1: Backend endpoint /api/qa/overview returns:**

```ts
{
  ncrOpen: 0,           // Phase 3 — return 0 stub now
  capaOpen: 0,          // Phase 3 — return 0 stub now
  overdueAudits: 0,     // Phase 4 — return 0 stub now
  stageBreaches: number, // count of orders past SLA per active stage
  pendingApprovals: number,
  pendingEsigs: number,  // unsigned acceptance records waiting for QA review
}
```

- [ ] **Step 2: Implement widgets (cards)** showing each metric with link-through

- [ ] **Step 3: RBAC: only `qa_manager` and `admin` see this dashboard**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/qa-dashboard/ apps/api/src/routes/qa-dashboard.ts
git commit -m "feat(phase2): qa_manager overview dashboard with stub NCR/CAPA cards"
```

---

### Week 11: SLA dashboard + access review + push notifications

#### Task 15: SLA breach service

**Files:**
- Create: `apps/api/src/services/slaBreachService.ts`
- Create: `apps/api/src/routes/sla-dashboard.ts`

- [ ] **Step 1: Stage SLA defined in stage_template (existing column from Phase 1: `target_duration_hours`)**

- [ ] **Step 2: Compute breach risk per active order:**

```ts
risk = elapsedInStage / targetDuration
status = risk >= 1.0 ? 'breached' : risk >= 0.8 ? 'at_risk' : 'on_track'
```

- [ ] **Step 3: GET /api/sla/dashboard returns list grouped by stage with counts**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/slaBreachService.ts apps/api/src/routes/sla-dashboard.ts
git commit -m "feat(phase2): SLA breach computation service + endpoint"
```

---

#### Task 16: SLA dashboard UI

**Files:**
- Create: `apps/web/src/features/sla-dashboard/SlaDashboardView.tsx`
- Create: `apps/web/src/features/sla-dashboard/BreachRiskTable.tsx`

- [ ] **Step 1: Stacked bar chart per stage (on_track / at_risk / breached)**

- [ ] **Step 2: Drill-down table with order ID, customer, current stage, hours overdue**

- [ ] **Step 3: Visible to admin, qa_manager, sales_manager (own team filter)**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/sla-dashboard/
git commit -m "feat(phase2): SLA dashboard UI with breach drill-down"
```

---

#### Task 17: Web Push subscription flow + VAPID setup

**Files:**
- Create: `apps/web/src/pwa/push-subscribe.ts`
- Create: `apps/api/src/routes/push.ts`
- Modify: `apps/api/src/services/notify.ts` (add Web Push channel)

- [ ] **Step 1: Generate VAPID keys, store in Cloudflare secrets**

```bash
bunx web-push generate-vapid-keys
wrangler secret put VAPID_PUBLIC_KEY
wrangler secret put VAPID_PRIVATE_KEY
```

- [ ] **Step 2: Subscribe flow**

```ts
export async function subscribePush() {
  const reg = await navigator.serviceWorker.ready;
  const sub = await reg.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
  });
  await api.post('/api/push/subscribe', sub.toJSON());
}
```

- [ ] **Step 3: API endpoint stores subscription in `push_subscriptions` table**

- [ ] **Step 4: Add Web Push adapter to notify service**

When alert fires for a user, fetch their push subscriptions and POST to endpoint with VAPID-signed payload. Use `web-push` library or raw VAPID JWT signing for Workers compatibility.

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/pwa/push-subscribe.ts apps/api/src/routes/push.ts apps/api/src/services/notify.ts
git commit -m "feat(phase2): Web Push subscription flow + VAPID + notify adapter"
```

---

#### Task 18: Access review — initiate + role/team change

**Files:**
- Create: `apps/api/src/services/accessReviewService.ts`
- Create: `apps/api/src/routes/access-review.ts`
- Create: `apps/web/src/features/access-review/AccessReviewView.tsx`
- Create: `apps/web/src/features/access-review/ReviewRoleAssignmentDialog.tsx`

- [ ] **Step 1: Write tests for service**

```ts
test('initiate creates review with one item per active user', async () => {
  await seedUsers(5, { disabled: false });
  const r = await initiateReview({ period: '2026-Q2', initiatedBy: 'admin1' });
  expect(r.itemCount).toBe(5);
});

test('cannot initiate two open reviews for same period', async () => {
  await initiateReview({ period: '2026-Q2', initiatedBy: 'admin1' });
  await expect(initiateReview({ period: '2026-Q2', initiatedBy: 'admin1' }))
    .rejects.toThrow(/already in progress/);
});
```

- [ ] **Step 2: Implement service**

```ts
export async function initiateReview({ period, initiatedBy }) {
  return db.transaction(async (tx) => {
    const existing = await tx.query.accessReviews.findFirst({
      where: and(eq(accessReviews.period, period), eq(accessReviews.status, 'in_progress'))
    });
    if (existing) throw new Error(`review for ${period} already in progress`);
    const id = ulid();
    await tx.insert(accessReviews).values({ id, period, initiatedBy, initiatedAt: new Date(), status: 'in_progress' });
    const users = await tx.select().from(usersTable).where(eq(usersTable.disabled, false));
    for (const u of users) {
      await tx.insert(accessReviewItems).values({
        id: ulid(), reviewId: id, userId: u.id,
        priorRole: u.role, priorTeamId: u.teamId,
        decision: 'pending', reason: '', reviewedAt: new Date(), reviewedBy: initiatedBy,
      });
    }
    await writeAudit(tx, { actor_user_id: initiatedBy, action: 'access_review.initiate', entity_type: 'access_review', entity_id: id });
    return { id, itemCount: users.length };
  });
}

export async function recordReviewDecision({ reviewId, itemId, decision, newRole, newTeamId, reason, reviewedBy }) {
  // Apply role/team change immediately (no batch — operator may change midway)
  await db.transaction(async (tx) => {
    const item = await tx.query.accessReviewItems.findFirst({ where: eq(accessReviewItems.id, itemId) });
    if (!item) throw new Error('item not found');
    await tx.update(accessReviewItems).set({ decision, newRole, newTeamId, reason, reviewedAt: new Date(), reviewedBy }).where(eq(accessReviewItems.id, itemId));
    if (decision === 'change_role' || decision === 'change_team') {
      const change: any = {};
      if (newRole) change.role = newRole;
      if (newTeamId !== undefined) change.teamId = newTeamId;
      await tx.update(usersTable).set(change).where(eq(usersTable.id, item.userId));
    } else if (decision === 'disable') {
      await tx.update(usersTable).set({ disabled: true }).where(eq(usersTable.id, item.userId));
      // Revoke all sessions
      await tx.delete(sessionsTable).where(eq(sessionsTable.userId, item.userId));
    }
    await writeAudit(tx, { actor_user_id: reviewedBy, action: 'access_review.decision', entity_type: 'user', entity_id: item.userId, detail_json: JSON.stringify({ decision, newRole, newTeamId, reason }) });
  });
}
```

- [ ] **Step 3: UI: AccessReviewView shows table of pending items, RoleAssignmentDialog applies change**

- [ ] **Step 4: RBAC: only `admin` can initiate / record decisions**

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/services/accessReviewService.ts apps/api/src/routes/access-review.ts apps/web/src/features/access-review/AccessReviewView.tsx apps/web/src/features/access-review/ReviewRoleAssignmentDialog.tsx
git commit -m "feat(phase2): access review — initiate + role/team change with audit"
```

---

#### Task 19: Access review sign-off (e-signature required)

**Files:**
- Create: `apps/web/src/features/access-review/ReviewSignOffDialog.tsx`
- Modify: `apps/api/src/services/accessReviewService.ts`

- [ ] **Step 1: Service: completeReview must require all items to have decision != 'pending'**

```ts
export async function completeReview({ reviewId, completedBy, esigToken, esigPassword, reason }) {
  return db.transaction(async (tx) => {
    const pending = await tx.select().from(accessReviewItems)
      .where(and(eq(accessReviewItems.reviewId, reviewId), eq(accessReviewItems.decision, 'pending')));
    if (pending.length > 0) throw new Error(`${pending.length} items still pending decision`);

    const review = await tx.query.accessReviews.findFirst({ where: eq(accessReviews.id, reviewId) });
    const allItems = await tx.select().from(accessReviewItems).where(eq(accessReviewItems.reviewId, reviewId));
    const recordHash = sha256(canonicalJson({ review, items: allItems }));

    const esig = await createESignature(tx, {
      userId: completedBy,
      recordType: 'access_review',
      recordId: reviewId,
      meaning: 'approval',
      reason,
      recordHash,
      reauthToken: esigToken,
      password: esigPassword,
    });

    await tx.update(accessReviews).set({
      status: 'completed', completedAt: new Date(), completedBy, signoffEsigId: esig.id
    }).where(eq(accessReviews.id, reviewId));

    await writeAudit(tx, { actor_user_id: completedBy, action: 'access_review.complete', entity_type: 'access_review', entity_id: reviewId, esig_id: esig.id });
  });
}
```

- [ ] **Step 2: UI: ReviewSignOffDialog requires re-auth password + reason text**

- [ ] **Step 3: Test: completeReview rejects when any item is pending**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/access-review/ReviewSignOffDialog.tsx apps/api/src/services/accessReviewService.ts
git commit -m "feat(phase2): access review sign-off with e-signature"
```

---

#### Task 20: Quarterly access review reminder cron

**Files:**
- Create: `apps/api/src/cron/access-review-reminder.ts`
- Modify: `apps/api/wrangler.toml` (add cron trigger)

- [ ] **Step 1: Cron: 1st of Jan/Apr/Jul/Oct at 09:00 → check if review for current quarter exists; if not, create alert + push notification to all admins**

- [ ] **Step 2: Add wrangler cron trigger**

```toml
[triggers]
crons = ["0 9 1 1,4,7,10 *", "0 * * * *"]   # quarterly + hourly alerts
```

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/cron/access-review-reminder.ts apps/api/wrangler.toml
git commit -m "feat(phase2): quarterly access review reminder cron"
```

---

### Week 12: Approval UI + retention + verification

#### Task 21: Dedicated approval queue UI (high-value orders)

**Files:**
- Create: `apps/web/src/features/approvals/ApprovalQueue.tsx`
- Modify: `apps/api/src/routes/orders.ts` (GET /api/approvals/pending)

- [ ] **Step 1: GET /api/approvals/pending returns orders awaiting approval (high-value or first-order from new customer)**

- [ ] **Step 2: UI: list with order summary, approve/reject buttons that trigger e-sig dialog**

- [ ] **Step 3: Visible to: admin, qa_manager, sales_manager (own team only)**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/approvals/ApprovalQueue.tsx apps/api/src/routes/orders.ts
git commit -m "feat(phase2): approval queue UI with e-sig flow"
```

---

#### Task 22: Push notification preference center

**Files:**
- Create: `apps/web/src/features/settings/NotificationSettings.tsx`
- Create: `apps/api/src/routes/notification-prefs.ts`

- [ ] **Step 1: User can toggle: stage advance, alert (own orders / team / all), access review reminder**

- [ ] **Step 2: Per-channel: email / web push / slack (if user has slack id linked)**

- [ ] **Step 3: Stored in `notification_prefs` table (JSON column on users may suffice)**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/settings/NotificationSettings.tsx apps/api/src/routes/notification-prefs.ts
git commit -m "feat(phase2): notification preference center"
```

---

#### Task 23: PQ-2 Phase 2 verification scenario

**Files:**
- Create: `apps/api/src/__tests__/pq2-mobile-flow.test.ts`

- [ ] **Step 1: Scenario covers**

1. Install PWA on mobile (manual verification step in test plan)
2. Ops scans GS1 barcode at incoming inspection → acceptance pass + audit
3. QA scans Lot at in-process QC → e-sig dialog → recorded with reason
4. QA scans Lot+Serial at final QC → e-sig → stage advances
5. Ops scans all units at shipment → e-sig → stage = packed_shipped
6. Sales_manager logs in → sees only own team's pipeline
7. Accountant marks deposit + final paid → stages advance, audit log entries
8. AR aging report shows zero outstanding
9. Admin initiates Q-review, decides for all 5 users (keep / change / disable), e-signs sign-off → review status = completed
10. SLA dashboard shows order completion within target

- [ ] **Step 2: Run scenario as integration test (mocked camera with seeded GS1 string)**

```bash
bun test apps/api/src/__tests__/pq2-mobile-flow.test.ts
```

Expected: all 10 sub-steps PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/__tests__/pq2-mobile-flow.test.ts
git commit -m "test(phase2): PQ-2 mobile + permissions verification scenario"
```

---

#### Task 24: Phase 2 deploy + CSP for camera + post-deploy smoke

**Files:**
- Modify: `apps/web/index.html` (CSP must allow `media-src 'self' blob:` and `camera` in Permissions-Policy)
- Modify: `apps/api/wrangler.toml` (deploy config check)

- [ ] **Step 1: Add headers in CF Pages `_headers` file**

```
/*
  Permissions-Policy: camera=(self), microphone=()
  Content-Security-Policy: default-src 'self'; img-src 'self' blob: data:; media-src 'self' blob:; connect-src 'self' https://api.orderflow.internal; script-src 'self'; style-src 'self' 'unsafe-inline';
```

- [ ] **Step 2: Deploy to staging**

```bash
cd apps/web && bun run build && bunx wrangler pages deploy dist --project-name orderflow-staging
cd ../api && bunx wrangler deploy --env staging
```

- [ ] **Step 3: Manual smoke on iOS Safari + Android Chrome**

- [ ] Install PWA → home-screen icon shows
- [ ] Camera permission prompt fires on first scan
- [ ] BarcodeDetector recognises a printed GS1 sample
- [ ] Push notification arrives on stage advance
- [ ] Offline: opening app shows cached shell, but submitting form fails with clear error (NetworkOnly enforced)

- [ ] **Step 4: Commit**

```bash
git add apps/web/_headers apps/api/wrangler.toml
git commit -m "chore(phase2): CSP/Permissions-Policy headers + post-deploy smoke checklist"
```

---

## Phase 2 Exit Criteria

- [ ] PWA installable on iOS Safari 16+ and Android Chrome
- [ ] GS1 barcode scanner reads GTIN/Lot/Serial/Expiry per UDI Rule
- [ ] Incoming, in-process, final QC, and shipment all support scan-driven acceptance
- [ ] In-process QC, final QC, shipment, release stages all require e-signature
- [ ] sales_manager sees only own team's pipeline
- [ ] accountant can mark payments; AR aging report works
- [ ] qa_manager sees overview dashboard (NCR/CAPA stubbed)
- [ ] Quarterly access review can be initiated, decisions recorded, signed off with e-sig
- [ ] SLA dashboard shows live breach risk per active order
- [ ] Web Push notifications deliver on iOS Safari 16.4+ and Android Chrome
- [ ] PQ-2 scenario passes end-to-end
- [ ] All audit log entries traceable; no orphan signatures
- [ ] Code review pass

**Approximate effort:** 4–5 weeks for one full-stack engineer

---

## Out of Scope (deferred)

- NCR / CAPA / Complaints / Document Control / Training Records (Phase 3)
- Annual QMS audit report generation (Phase 4)
- MCP server for Teams Copilot (Phase 4)
- Software validation IQ/OQ/PQ formal protocols (Phase 4)
- RFC 3161 trusted timestamping (Phase 4)
- Disaster recovery drill (Phase 4)
- Native mobile app (out of scope entirely; PWA is the strategy)
