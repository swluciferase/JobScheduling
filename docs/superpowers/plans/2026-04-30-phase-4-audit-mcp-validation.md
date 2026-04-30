# OrderFlow Phase 4: Audit Reports + Internal MCP + Software Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the regulatory-readiness layer: (1) annual QMS audit report PDF generator that auto-compiles a full audit package from system data; (2) internal Model Context Protocol (MCP) server exposing OrderFlow QMS data to Microsoft Teams Copilot for natural-language query (read-only by default; write operations require interactive e-sig in the web UI, never via Copilot); (3) formal software validation per 21 CFR 820.70(i) — IQ/OQ/PQ protocols, validation master plan, traceability matrix; (4) RFC 3161 trusted timestamping for the audit chain + e-sig hashes; (5) management review minutes generator (820.20); (6) disaster recovery drill protocol.

**Architecture:** Add four vertical slices on top of Phase 0–3:
- A `reports/` service that templates audit packages and management review minutes into PDF (using `@react-pdf/renderer` for templated layouts) with embedded e-sig manifestations, then mirrors the final PDF to R2 immutable bucket and registers an RFC 3161 timestamp via FreeTSA or equivalent.
- A separate `apps/mcp` Worker hosting MCP HTTP+SSE transport, authenticated via OAuth 2.0 Authorization Code + PKCE behind Cloudflare Access (Zero Trust gates Copilot's outbound traffic the same way it gates web users); MCP tools call existing services through `packages/shared`.
- A `validation/` package containing the VMP markdown source, IQ/OQ/PQ test scripts (re-using PQ-1/2/3 scenarios from earlier phases as PQ artifacts), and a traceability matrix generator that maps each spec requirement to test result.
- A timestamping service that batches audit_log seq ranges hourly and submits the Merkle root of the range's hashes to a TSA, storing the resulting RFC 3161 timestamp token alongside the seq range.

**Tech Stack:** Same as Phase 1–3 + `@react-pdf/renderer` (PDF templating), `node-rfc3161` or hand-rolled ASN.1/CMS handling for FreeTSA (Workers-compatible — small footprint, no Node deps), `@modelcontextprotocol/sdk` (MCP TypeScript SDK), `markdown-it` (render VMP/IQ/OQ markdown for printable PDFs).

**Phase boundary:** Ends when (a) admin can run the annual audit report generator that produces a 100+ page signed PDF covering all 9 QMS audit areas, stored immutably with RFC 3161 timestamp; (b) Teams Copilot user can ask "Show me open NCRs over 30 days old" and get an answer via MCP without leaving Teams; (c) admin sees a Validation tab showing VMP version, IQ/OQ/PQ test results, traceability matrix coverage, and re-validation triggers; (d) audit log entries newer than 24h have an associated RFC 3161 timestamp token; (e) admin has run the documented DR drill at least once and the result PDF is stored. Risk Management File (ISO 14971) remains out of scope as a separate project.

---

## Reference Spec

Cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §3 Regulatory mapping (820.20 management review, 820.22 internal audit, 820.70(i) software validation, 21 CFR Part 11 11.10(a) closed system validation)
- §15 Software validation (VMP / IQ / OQ / PQ)
- §17 Audit log + RFC 3161 trusted timestamping
- §19 Disaster recovery + business continuity
- §20 MCP server scope (read-only by default, never writes through Copilot)

---

## File Structure

```
orderflow/
├── apps/
│   ├── mcp/                                   # NEW — Worker for MCP server
│   │   ├── src/
│   │   │   ├── index.ts                       # Worker entry
│   │   │   ├── mcp-server.ts                  # MCP transport setup
│   │   │   ├── oauth/
│   │   │   │   ├── handler.ts                 # OAuth Authorization Code + PKCE
│   │   │   │   ├── token-introspect.ts
│   │   │   │   └── pkce.ts
│   │   │   ├── tools/                         # Read-only MCP tools
│   │   │   │   ├── orders.ts                  # query_orders, get_order
│   │   │   │   ├── ncrs.ts                    # query_ncrs
│   │   │   │   ├── capas.ts                   # query_capas
│   │   │   │   ├── complaints.ts
│   │   │   │   ├── documents.ts               # find_document
│   │   │   │   ├── training.ts                # check_training_status
│   │   │   │   ├── audit.ts                   # search_audit_log (admin-only)
│   │   │   │   └── reports.ts                 # generate_management_summary
│   │   │   └── audit.ts                       # All MCP calls audit-logged
│   │   └── wrangler.toml
│   ├── api/
│   │   └── src/
│   │       ├── routes/
│   │       │   ├── audit-reports.ts            # POST /api/audit-reports/annual
│   │       │   ├── management-review.ts
│   │       │   ├── validation.ts               # GET /api/validation/status
│   │       │   ├── timestamps.ts               # GET /api/timestamps/:seqRange
│   │       │   └── dr.ts                       # POST /api/dr/drill-result
│   │       ├── services/
│   │       │   ├── auditReportBuilder.ts
│   │       │   ├── managementReviewBuilder.ts
│   │       │   ├── traceabilityMatrixService.ts
│   │       │   ├── rfc3161Service.ts
│   │       │   └── drDrillService.ts
│   │       ├── reports/
│   │       │   ├── AnnualAuditReport.tsx       # @react-pdf/renderer template
│   │       │   ├── ManagementReviewMinutes.tsx
│   │       │   └── components/
│   │       │       ├── PageHeader.tsx
│   │       │       ├── ESigManifestationBlock.tsx
│   │       │       ├── DataTable.tsx
│   │       │       └── TimestampFooter.tsx
│   │       ├── cron/
│   │       │   ├── timestamp-batch.ts          # Hourly RFC 3161 batch
│   │       │   └── revalidation-check.ts
│   │       └── db/
│   │           └── migrations/
│   │               └── 0009_audit_validation.sql
└── packages/
    └── validation/                              # NEW — formal validation artifacts
        ├── vmp/
        │   ├── VMP-001-orderflow.md             # Validation Master Plan
        │   └── change-control-procedure.md
        ├── iq/
        │   ├── IQ-001-cloudflare-environment.md
        │   └── IQ-002-d1-r2-config.md
        ├── oq/
        │   ├── OQ-001-rbac.md
        │   ├── OQ-002-stage-engine.md
        │   ├── OQ-003-esig-framework.md
        │   ├── OQ-004-audit-chain.md
        │   └── OQ-005-document-control.md
        ├── pq/
        │   ├── PQ-001-mvp-flow.md               # → reuses Phase 1 pq1 test
        │   ├── PQ-002-mobile-flow.md            # → reuses Phase 2 pq2 test
        │   └── PQ-003-qms-flow.md               # → reuses Phase 3 pq3 test
        ├── traceability/
        │   ├── matrix-generator.ts              # Spec section ↔ test ↔ result
        │   └── matrix.csv                       # Output (committed)
        └── package.json
```

---

## Task List

### Week 19: RFC 3161 timestamping + audit report data model

#### Task 1: DB migration for timestamps + audit reports + DR drills

**Files:**
- Create: `apps/api/src/db/migrations/0009_audit_validation.sql`
- Modify: `apps/api/src/db/schema.ts`

- [ ] **Step 1: Write migration**

```sql
-- RFC 3161 timestamp tokens covering ranges of audit_log seq
CREATE TABLE audit_timestamp_tokens (
  id TEXT PRIMARY KEY,
  seq_range_start INTEGER NOT NULL,
  seq_range_end INTEGER NOT NULL,
  merkle_root TEXT NOT NULL,                    -- hex sha-256 of leaves
  tsa_url TEXT NOT NULL,
  tsa_token_b64 TEXT NOT NULL,                  -- DER-encoded TimeStampToken in base64
  tsa_serial TEXT,
  tsa_time TIMESTAMP NOT NULL,                  -- the time TSA assigned
  request_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  verified_at TIMESTAMP,                        -- last successful verification
  status TEXT NOT NULL DEFAULT 'issued'         -- issued | verified | broken
);
CREATE INDEX idx_timestamps_range ON audit_timestamp_tokens(seq_range_start, seq_range_end);
CREATE INDEX idx_timestamps_status ON audit_timestamp_tokens(status);

-- Annual / ad-hoc QMS audit reports
CREATE TABLE audit_reports (
  id TEXT PRIMARY KEY,
  report_number TEXT NOT NULL UNIQUE,           -- AR-2026-001
  report_type TEXT NOT NULL,                    -- annual | internal_qa_audit | external_response | mock_audit
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  generated_by TEXT NOT NULL REFERENCES users(id),
  generated_at TIMESTAMP NOT NULL,
  pdf_r2_key TEXT NOT NULL,                     -- in immutable bucket
  pdf_sha256 TEXT NOT NULL,
  signoff_esig_id TEXT REFERENCES e_signatures(id),
  signoff_at TIMESTAMP,
  rfc3161_token_id TEXT REFERENCES audit_timestamp_tokens(id),
  status TEXT NOT NULL DEFAULT 'draft',         -- draft | signed | archived
  metrics_json TEXT,                            -- structured snapshot used to render
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Management review minutes (820.20)
CREATE TABLE management_reviews (
  id TEXT PRIMARY KEY,
  review_number TEXT NOT NULL UNIQUE,           -- MR-2026-001
  scheduled_date DATE NOT NULL,
  conducted_at TIMESTAMP,
  attendees_json TEXT NOT NULL,                 -- JSON array of user_ids + names
  inputs_summary TEXT,                          -- TipTap HTML
  outputs_summary TEXT,                         -- TipTap HTML
  action_items_json TEXT,                       -- structured list with owner+date
  pdf_r2_key TEXT,
  signoff_esig_id TEXT REFERENCES e_signatures(id),
  status TEXT NOT NULL DEFAULT 'scheduled'      -- scheduled | conducted | signed
);

-- Disaster recovery drill log
CREATE TABLE dr_drills (
  id TEXT PRIMARY KEY,
  drill_number TEXT NOT NULL UNIQUE,            -- DR-2026-001
  scenario TEXT NOT NULL,                       -- d1_corruption | r2_outage | full_region_failure | secret_compromise
  initiated_by TEXT NOT NULL REFERENCES users(id),
  initiated_at TIMESTAMP NOT NULL,
  rto_target_minutes INTEGER NOT NULL,
  rpo_target_minutes INTEGER NOT NULL,
  rto_actual_minutes INTEGER,
  rpo_actual_minutes INTEGER,
  result TEXT NOT NULL DEFAULT 'in_progress',   -- in_progress | passed | failed
  findings TEXT,
  follow_up_capa_id TEXT,
  signoff_esig_id TEXT REFERENCES e_signatures(id),
  drill_report_pdf_r2_key TEXT
);

-- Validation status snapshots
CREATE TABLE validation_runs (
  id TEXT PRIMARY KEY,
  protocol_code TEXT NOT NULL,                  -- VMP | IQ-001 | OQ-001 | PQ-001 etc.
  protocol_version TEXT NOT NULL,
  executed_by TEXT NOT NULL REFERENCES users(id),
  executed_at TIMESTAMP NOT NULL,
  result TEXT NOT NULL,                         -- pass | fail | partial
  notes TEXT,
  evidence_doc_id TEXT REFERENCES documents(id),
  signoff_esig_id TEXT REFERENCES e_signatures(id),
  triggered_by TEXT                             -- initial | change_request | scheduled
);
CREATE INDEX idx_validation_runs_protocol ON validation_runs(protocol_code);
```

- [ ] **Step 2: Update Drizzle schema**

- [ ] **Step 3: Apply migration**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/db/migrations/0009_audit_validation.sql apps/api/src/db/schema.ts
git commit -m "feat(phase4): schema for timestamps, audit_reports, management_reviews, dr_drills, validation_runs"
```

---

#### Task 2: RFC 3161 client (FreeTSA, Workers-compatible)

**Files:**
- Create: `apps/api/src/services/rfc3161Service.ts`
- Create: `apps/api/src/services/__tests__/rfc3161Service.test.ts`

- [ ] **Step 1: Write failing tests**

```ts
test('builds TSA request DER for SHA-256 hash', () => {
  const hash = '0123456789abcdef'.repeat(4);   // 64-char hex
  const der = buildTsaRequest(hash);
  expect(der).toBeInstanceOf(Uint8Array);
  expect(der[0]).toBe(0x30); // SEQUENCE
});

test('parses TimeStampToken response and extracts time', async () => {
  // Use a fixture token from FreeTSA test endpoint
  const tokenB64 = await fixtures.readBase64('tsa-response-fixture.tst');
  const parsed = parseTsaResponse(tokenB64);
  expect(parsed.serial).toMatch(/^[0-9a-f]+$/);
  expect(parsed.time).toBeInstanceOf(Date);
});
```

- [ ] **Step 2: Implement minimal ASN.1/CMS handling**

```ts
// We only need:
// - Build a TimeStampReq (RFC 3161 §2.4.1) with imprint = SHA-256 hash
// - Parse the TimeStampResp (RFC 3161 §2.4.2) for status + token
// - Extract genTime from TSTInfo inside the token
// We don't validate signature here — that's done by verifyTimestamp via WebCrypto.

import { encodeAsn1, decodeAsn1 } from './asn1';

const TS_REQ_OID_SHA256 = '2.16.840.1.101.3.4.2.1';

export function buildTsaRequest(hexHash: string): Uint8Array {
  const hashBytes = hexToBytes(hexHash);
  // SEQUENCE { version=1, MessageImprint { hashAlg, hashedMessage }, certReq=true }
  return encodeAsn1.SEQUENCE([
    encodeAsn1.INTEGER(1),
    encodeAsn1.SEQUENCE([
      encodeAsn1.SEQUENCE([encodeAsn1.OID(TS_REQ_OID_SHA256), encodeAsn1.NULL()]),
      encodeAsn1.OCTET_STRING(hashBytes),
    ]),
    encodeAsn1.BOOLEAN(true),
  ]);
}

export interface ParsedTimestamp {
  serial: string;
  time: Date;
  tokenB64: string;
}

export function parseTsaResponse(b64: string): ParsedTimestamp {
  // ... parse TimeStampResp → status, TimeStampToken → SignedData → TSTInfo
}

export async function requestTimestamp(merkleRoot: string, tsaUrl: string): Promise<ParsedTimestamp> {
  const der = buildTsaRequest(merkleRoot);
  const res = await fetch(tsaUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/timestamp-query' },
    body: der,
  });
  if (!res.ok) throw new Error(`TSA HTTP ${res.status}`);
  const respBytes = new Uint8Array(await res.arrayBuffer());
  const tokenB64 = btoa(String.fromCharCode(...respBytes));
  return parseTsaResponse(tokenB64);
}
```

- [ ] **Step 3: Verify tests pass**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/rfc3161Service.ts apps/api/src/services/__tests__/rfc3161Service.test.ts apps/api/src/services/asn1.ts
git commit -m "feat(phase4): RFC 3161 client — build TSA request, parse TimeStampToken (Workers-compatible)"
```

---

#### Task 3: Hourly timestamp batch cron

**Files:**
- Create: `apps/api/src/cron/timestamp-batch.ts`
- Modify: `apps/api/wrangler.toml`

- [ ] **Step 1: Logic**

1. Query last timestamped seq (MAX(seq_range_end) from audit_timestamp_tokens; 0 if empty).
2. Query audit_log entries with seq > last_timestamped, ordered by seq, capped at 10000 per run.
3. If no new entries, exit.
4. Compute Merkle root of {row_hash} for the batch (sha-256 binary tree).
5. Call requestTimestamp(merkleRoot, FREETSA_URL).
6. Insert audit_timestamp_tokens row with seq_range_start, seq_range_end, merkle_root, tsa_token_b64, tsa_time.

- [ ] **Step 2: Implementation**

```ts
export async function runTimestampBatch(env: Env) {
  const last = await env.DB.prepare(`SELECT MAX(seq_range_end) AS last FROM audit_timestamp_tokens`).first();
  const lastSeq = (last?.last as number) ?? 0;

  const rows = await env.DB.prepare(`
    SELECT seq, row_hash FROM audit_log WHERE seq > ? ORDER BY seq LIMIT 10000
  `).bind(lastSeq).all();

  if (rows.results.length === 0) return;

  const leaves = rows.results.map((r: any) => hexToBytes(r.row_hash));
  const root = bytesToHex(await merkleRoot(leaves));

  const ts = await requestTimestamp(root, env.RFC3161_TSA_URL);
  const seqStart = (rows.results[0] as any).seq;
  const seqEnd = (rows.results[rows.results.length - 1] as any).seq;

  await env.DB.prepare(`
    INSERT INTO audit_timestamp_tokens
      (id, seq_range_start, seq_range_end, merkle_root, tsa_url, tsa_token_b64, tsa_serial, tsa_time, status)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'issued')
  `).bind(ulid(), seqStart, seqEnd, root, env.RFC3161_TSA_URL, ts.tokenB64, ts.serial, ts.time.toISOString()).run();
}
```

- [ ] **Step 3: Add to wrangler.toml cron**

```toml
[triggers]
crons = ["0 * * * *", "0 6 * * *", "0 9 1 1,4,7,10 *"]   # hourly TSA + daily training + quarterly review
```

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/cron/timestamp-batch.ts apps/api/wrangler.toml
git commit -m "feat(phase4): hourly RFC 3161 timestamp batch over audit_log Merkle root"
```

---

#### Task 4: Timestamp verification endpoint

**Files:**
- Create: `apps/api/src/routes/timestamps.ts`

- [ ] **Step 1: GET /api/timestamps/verify/:seqRangeStart/:seqRangeEnd**

Verifies that the stored Merkle root matches a fresh recompute of audit_log[start..end]. Updates verified_at on success, marks 'broken' on mismatch.

- [ ] **Step 2: GET /api/timestamps/list?from=&to= for admin viewing**

- [ ] **Step 3: Tests for tamper detection**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/routes/timestamps.ts
git commit -m "feat(phase4): timestamp verification + listing endpoints"
```

---

### Week 20: Annual audit report generator

#### Task 5: Audit report data collector

**Files:**
- Create: `apps/api/src/services/auditReportBuilder.ts`

- [ ] **Step 1: Collector aggregates 9 audit areas**

```ts
export async function collectAnnualMetrics(periodStart: Date, periodEnd: Date) {
  return {
    coverPage: { reportNumber, periodStart, periodEnd, generatedAt, organization },
    section1_QMSOverview: await collectQmsOverview(),
    section2_DesignControls: await collectDesignControls(periodStart, periodEnd),  // 820.30 — out of full scope, links to design history file
    section3_DocumentControl: await collectDocumentControl(periodStart, periodEnd),  // 820.40 — versions, approvals, acks
    section4_PurchasingControls: await collectPurchasingControls(periodStart, periodEnd),  // 820.50 — supplier UDIs, incoming inspection results
    section5_ProductionAndProcessControls: await collectProduction(periodStart, periodEnd),  // 820.70 — DHRs, in-process QC
    section6_AcceptanceActivities: await collectAcceptance(periodStart, periodEnd),  // 820.80 — incoming/in-process/final QC stats
    section7_NCRAndCAPA: await collectNcrCapa(periodStart, periodEnd),  // 820.90/100
    section8_ComplaintsAndMDR: await collectComplaints(periodStart, periodEnd),  // 820.198
    section9_TrainingAndAccess: await collectTrainingAccess(periodStart, periodEnd),  // 820.25 + access reviews
    section10_AuditTrailIntegrity: await collectAuditIntegrity(periodStart, periodEnd),  // hash chain + RFC 3161 verify summary
    section11_ManagementReviews: await collectManagementReviews(periodStart, periodEnd),  // 820.20
    appendixA_ESignatures: await collectAllESigs(periodStart, periodEnd),
    appendixB_TimestampTokens: await collectTimestampTokens(periodStart, periodEnd),
  };
}
```

- [ ] **Step 2: Each collector returns structured object that maps directly to PDF section**

- [ ] **Step 3: Run all collectors in transaction with READ-only consistency**

- [ ] **Step 4: Tests with seeded year of data**

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/services/auditReportBuilder.ts apps/api/src/services/__tests__/auditReportBuilder.test.ts
git commit -m "feat(phase4): annual audit report data collector — 9 sections + 2 appendices"
```

---

#### Task 6: PDF template — AnnualAuditReport (react-pdf)

**Files:**
- Create: `apps/api/src/reports/AnnualAuditReport.tsx`
- Create: `apps/api/src/reports/components/`

- [ ] **Step 1: Install dependencies**

```bash
cd apps/api && bun add @react-pdf/renderer
```

- [ ] **Step 2: Cover page + table of contents + each section as separate Page component**

- [ ] **Step 3: ESigManifestationBlock renders Part 11-compliant signature display**

```tsx
export function ESigManifestationBlock({ esig }: { esig: ESignature }) {
  return (
    <View style={styles.sigBlock}>
      <Text>Signed by: {esig.user_name_snapshot}</Text>
      <Text>Role at signing: {esig.user_role_snapshot}</Text>
      <Text>Date/Time: {esig.signed_at} (UTC)</Text>
      <Text>Meaning: {esig.meaning}</Text>
      <Text>Reason: {esig.reason ?? '—'}</Text>
      <Text>Audit ID: {esig.audit_id}</Text>
      <Text>Record hash: {truncate(esig.record_hash, 32)}…</Text>
    </View>
  );
}
```

- [ ] **Step 4: Tables (DataTable component) for tabular data**

- [ ] **Step 5: Pages numbered N of M; each page footer includes report_number + RFC 3161 timestamp serial**

- [ ] **Step 6: Visual smoke render to disk**

- [ ] **Step 7: Commit**

```bash
git add apps/api/src/reports/AnnualAuditReport.tsx apps/api/src/reports/components/
git commit -m "feat(phase4): annual audit report PDF template (react-pdf)"
```

---

#### Task 7: Audit report generation route + sign-off

**Files:**
- Create: `apps/api/src/routes/audit-reports.ts`

- [ ] **Step 1: POST /api/audit-reports/annual** (admin only)

```ts
// 1. Collect metrics
// 2. Render PDF (Buffer)
// 3. Compute SHA-256 of PDF
// 4. Upload to R2 immutable bucket: audit-reports/{reportNumber}.pdf (no-delete lifecycle)
// 5. Insert audit_reports row, status = 'draft'
// 6. Return { reportId, downloadUrl }
```

- [ ] **Step 2: POST /api/audit-reports/:id/signoff** (requires admin + e-sig + reason)

- [ ] **Step 3: After sign-off, queue immediate timestamp request for the PDF SHA-256 (in addition to the routine batch)**

- [ ] **Step 4: Tests**

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/routes/audit-reports.ts
git commit -m "feat(phase4): audit report generation + sign-off route with R2 + RFC 3161"
```

---

#### Task 8: Audit report UI

**Files:**
- Create: `apps/web/src/features/audit-reports/AuditReportListView.tsx`
- Create: `apps/web/src/features/audit-reports/GenerateReportDialog.tsx`
- Create: `apps/web/src/features/audit-reports/SignOffDialog.tsx`

- [ ] **Step 1: List all generated reports with status badge, signed-by, RFC 3161 verify state**

- [ ] **Step 2: Generate dialog: select period (default = previous calendar year), confirm**

- [ ] **Step 3: Sign-off dialog: e-sig flow + reason text**

- [ ] **Step 4: Inline PDF preview after generation**

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/features/audit-reports/
git commit -m "feat(phase4): audit report UI — list, generate, sign-off"
```

---

### Week 21: Management review minutes + DR drill

#### Task 9: Management review minutes generator (820.20)

**Files:**
- Create: `apps/api/src/services/managementReviewBuilder.ts`
- Create: `apps/api/src/routes/management-review.ts`
- Create: `apps/api/src/reports/ManagementReviewMinutes.tsx`

- [ ] **Step 1: Required management review inputs per 820.20(c)**

```ts
{
  qualityPolicyAndObjectives: { current, kpis_vs_target },
  customerFeedback: { complaintsCount, severity, mdrFiled, satisfactionScore },
  productPerformance: { ncrTrends, capaEffectiveness, fieldFailures },
  processPerformance: { stageBreachStats, slaCompliance },
  internalAudit: { findingsClosedRate, openFindingsByAge },
  followUpFromPriorReviews: [...]
}
```

- [ ] **Step 2: Required outputs per 820.20(c)(2)**

```ts
{
  improvementsToQms: [...],
  improvementsToProduct: [...],
  resourceNeeds: [...],
  followUpActionItems: [{ description, owner, dueDate }]
}
```

- [ ] **Step 3: UI: schedule meeting → conduct (TipTap editor for inputs/outputs) → action items → sign-off (e-sig)**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/managementReviewBuilder.ts apps/api/src/routes/management-review.ts apps/api/src/reports/ManagementReviewMinutes.tsx
git commit -m "feat(phase4): management review minutes generator (820.20)"
```

---

#### Task 10: Management review UI

**Files:**
- Create: `apps/web/src/features/management-review/MrListView.tsx`
- Create: `apps/web/src/features/management-review/MrConductView.tsx`
- Create: `apps/web/src/features/management-review/MrSignOffDialog.tsx`

- [ ] **Step 1: Schedule + conduct flow**

- [ ] **Step 2: Auto-populate inputs section with current metrics on open**

- [ ] **Step 3: Action items emit follow-ups visible in management review dashboard**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/management-review/
git commit -m "feat(phase4): management review UI — schedule, conduct, sign-off"
```

---

#### Task 11: DR drill protocol + result capture

**Files:**
- Create: `apps/api/src/services/drDrillService.ts`
- Create: `apps/api/src/routes/dr.ts`
- Create: `packages/validation/dr/DR-PROCEDURE-001.md`

- [ ] **Step 1: Author DR-PROCEDURE-001.md describing 4 scenarios**

1. D1 corruption → restore from D1 PITR (point-in-time recovery)
2. R2 outage → switch reads to fallback bucket; verify regular bucket recovery
3. Full Cloudflare account compromise → rotate all secrets + revoke all sessions
4. Loss of e-sig signing key (would-be future) → rotate KMS key + re-sign baseline

- [ ] **Step 2: Service: createDrill / recordDrillResult / signDrillReport**

```ts
export async function recordDrillResult({ drillId, rtoActual, rpoActual, result, findings, executedBy, esigToken, esigPassword, reason, followUpCapaId }) {
  return db.transaction(async (tx) => {
    const drill = await tx.query.drDrills.findFirst({ where: eq(drDrills.id, drillId) });
    if (!drill) throw new Error('not found');

    const recordHash = sha256(canonicalJson({ drillId, result, rtoActual, rpoActual, findings }));
    const esig = await createESignature(tx, {
      userId: executedBy, recordType: 'dr_drill', recordId: drillId,
      meaning: 'review', reason, recordHash,
      reauthToken: esigToken, password: esigPassword,
    });

    await tx.update(drDrills).set({
      rtoActualMinutes: rtoActual, rpoActualMinutes: rpoActual,
      result, findings, followUpCapaId, signoffEsigId: esig.id,
    }).where(eq(drDrills.id, drillId));

    await writeAudit(tx, { actor_user_id: executedBy, action: 'dr_drill.complete', entity_type: 'dr_drill', entity_id: drillId, esig_id: esig.id });
  });
}
```

- [ ] **Step 3: PDF report renders results + e-sig manifestation; uploaded to R2 immutable**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/drDrillService.ts apps/api/src/routes/dr.ts packages/validation/dr/DR-PROCEDURE-001.md
git commit -m "feat(phase4): DR drill service + 4 documented scenarios + report"
```

---

#### Task 12: DR drill UI

**Files:**
- Create: `apps/web/src/features/dr/DrillListView.tsx`
- Create: `apps/web/src/features/dr/DrillRunView.tsx`

- [ ] **Step 1: List of past + scheduled drills with RTO/RPO target vs actual**

- [ ] **Step 2: Drill run view: scenario selector, timer, evidence upload, sign-off**

- [ ] **Step 3: If failed → CTA "Open CAPA" pre-fills with drill metadata**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/dr/
git commit -m "feat(phase4): DR drill UI — list + run + sign-off"
```

---

### Week 22: Software validation (VMP/IQ/OQ/PQ + traceability matrix)

#### Task 13: Author VMP + IQ + OQ + PQ markdown protocols

**Files:**
- Create: `packages/validation/vmp/VMP-001-orderflow.md`
- Create: `packages/validation/iq/IQ-001-cloudflare-environment.md`
- Create: `packages/validation/iq/IQ-002-d1-r2-config.md`
- Create: `packages/validation/oq/OQ-001-rbac.md`
- Create: `packages/validation/oq/OQ-002-stage-engine.md`
- Create: `packages/validation/oq/OQ-003-esig-framework.md`
- Create: `packages/validation/oq/OQ-004-audit-chain.md`
- Create: `packages/validation/oq/OQ-005-document-control.md`
- Create: `packages/validation/pq/PQ-001-mvp-flow.md`
- Create: `packages/validation/pq/PQ-002-mobile-flow.md`
- Create: `packages/validation/pq/PQ-003-qms-flow.md`

- [ ] **Step 1: VMP defines:**

- Scope (system, modules, environment)
- Risk classification (per GAMP 5: this is a Category 4 configured product hosting Category 5 custom code)
- Validation strategy (IQ for environment, OQ per module, PQ for end-to-end scenarios)
- Roles & responsibilities (validation lead, executor, approver per SoD-4)
- Change control trigger (any code change to a validated module → re-execute affected OQ + PQ before release)
- Acceptance criteria
- Re-validation schedule (annual at minimum, plus on-change)

- [ ] **Step 2: Each IQ/OQ/PQ document follows template**

```
1. Purpose
2. Scope
3. Prerequisites
4. Test Steps (numbered)
5. Expected Results
6. Acceptance Criteria
7. Deviation Handling
8. Approval Block (executed_by + e-sig + date)
```

- [ ] **Step 3: PQ-001/002/003 reference the existing test scripts**

```
PQ-001 corresponds to apps/api/src/__tests__/pq1-class2-order-flow.test.ts (Phase 1)
PQ-002 corresponds to apps/api/src/__tests__/pq2-mobile-flow.test.ts (Phase 2)
PQ-003 corresponds to apps/api/src/__tests__/pq3-qms-flow.test.ts (Phase 3)
```

- [ ] **Step 4: Each PQ defines manual smoke steps that the automated test alone doesn't cover (mobile install, push delivery, browser camera permission)**

- [ ] **Step 5: Commit**

```bash
git add packages/validation/vmp/ packages/validation/iq/ packages/validation/oq/ packages/validation/pq/
git commit -m "feat(phase4): authored VMP + IQ-001/002 + OQ-001..005 + PQ-001/002/003"
```

---

#### Task 14: Validation runs API + recording

**Files:**
- Create: `apps/api/src/routes/validation.ts`
- Modify: `apps/api/src/services/validationService.ts`

- [ ] **Step 1: POST /api/validation/runs** records protocol execution result

```ts
{
  protocolCode: 'OQ-002',
  protocolVersion: 'v1.0',
  result: 'pass',
  notes: '...',
  evidenceDocId: '...',
  triggeredBy: 'change_request',
  esigToken, esigPassword, reason
}
```

- [ ] **Step 2: GET /api/validation/status returns matrix**

```ts
{
  vmpVersion: 'v1.0',
  protocols: [
    { code: 'IQ-001', version: 'v1.0', lastRun: '...', result: 'pass', dueDate: 'YYYY-MM-DD' },
    ...
  ],
  overdue: [...],
  changeControlOpen: 0
}
```

- [ ] **Step 3: Tests**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/routes/validation.ts apps/api/src/services/validationService.ts
git commit -m "feat(phase4): validation runs API + status matrix"
```

---

#### Task 15: Traceability matrix generator

**Files:**
- Create: `packages/validation/traceability/matrix-generator.ts`
- Create: `packages/validation/traceability/matrix.csv`

- [ ] **Step 1: Generator parses spec markdown for requirement IDs (e.g. `[REQ-040-01]` per section)**

- [ ] **Step 2: Cross-references with OQ/PQ test step labels (e.g. `[TEST-OQ-002-04]`)**

- [ ] **Step 3: Outputs CSV: requirement_id, requirement_text, test_protocols, last_test_date, last_result, status**

- [ ] **Step 4: CI step: regenerate matrix on every change to spec or validation/**

```bash
bun run packages/validation/traceability/matrix-generator.ts > packages/validation/traceability/matrix.csv
```

- [ ] **Step 5: Commit**

```bash
git add packages/validation/traceability/
git commit -m "feat(phase4): traceability matrix generator + initial CSV"
```

---

#### Task 16: Validation tab UI

**Files:**
- Create: `apps/web/src/features/validation/ValidationStatusView.tsx`
- Create: `apps/web/src/features/validation/RunProtocolDialog.tsx`
- Create: `apps/web/src/features/validation/TraceabilityMatrixView.tsx`

- [ ] **Step 1: Status view: matrix of protocols, last run, result, due date with red/amber/green status**

- [ ] **Step 2: RunProtocolDialog: protocol picker, paste result, evidence upload, e-sig**

- [ ] **Step 3: TraceabilityMatrixView: searchable/filterable table with deep-links to each protocol document**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/validation/
git commit -m "feat(phase4): validation tab UI — status, run, traceability matrix"
```

---

### Week 23: Internal MCP server (read-only Teams Copilot integration)

#### Task 17: MCP Worker scaffold

**Files:**
- Create: `apps/mcp/wrangler.toml`
- Create: `apps/mcp/src/index.ts`
- Create: `apps/mcp/package.json`

- [ ] **Step 1: New Worker at `apps/mcp` with separate wrangler config**

```toml
name = "orderflow-mcp"
main = "src/index.ts"
compatibility_date = "2026-04-01"

[[d1_databases]]
binding = "DB"
database_name = "orderflow-prod"
database_id = "..."

[[r2_buckets]]
binding = "DOCUMENTS"
bucket_name = "orderflow-documents"

[vars]
MCP_BASE_URL = "https://mcp.orderflow.internal"
ENTRA_TENANT_ID = "..."
```

- [ ] **Step 2: Install MCP SDK**

```bash
cd apps/mcp && bun add @modelcontextprotocol/sdk
```

- [ ] **Step 3: Hello-world MCP server with /sse and /messages endpoints**

- [ ] **Step 4: Cloudflare Access policy gates this Worker (same Zero Trust posture as web)**

- [ ] **Step 5: Commit**

```bash
git add apps/mcp/
git commit -m "feat(phase4): MCP Worker scaffold + Cloudflare Access gating"
```

---

#### Task 18: OAuth Authorization Code + PKCE handler

**Files:**
- Create: `apps/mcp/src/oauth/handler.ts`
- Create: `apps/mcp/src/oauth/pkce.ts`
- Create: `apps/mcp/src/oauth/token-introspect.ts`

- [ ] **Step 1: Authorize endpoint: redirects to Entra OIDC for user consent**

- [ ] **Step 2: Callback exchanges authorization_code for tokens**

- [ ] **Step 3: Issue MCP-scoped access tokens (short-lived, ≤30 min) bound to user_id, role, and consented scopes**

- [ ] **Step 4: Token introspection on every MCP request to verify session not revoked**

- [ ] **Step 5: All MCP scopes are read-only: `read:orders`, `read:ncrs`, `read:capas`, `read:complaints`, `read:documents`, `read:training`, `read:audit` (admin only)**

- [ ] **Step 6: Commit**

```bash
git add apps/mcp/src/oauth/
git commit -m "feat(phase4): OAuth Auth Code + PKCE for MCP — read-only scopes"
```

---

#### Task 19: Read-only MCP tools — orders, ncrs, capas, complaints

**Files:**
- Create: `apps/mcp/src/tools/orders.ts`
- Create: `apps/mcp/src/tools/ncrs.ts`
- Create: `apps/mcp/src/tools/capas.ts`
- Create: `apps/mcp/src/tools/complaints.ts`

- [ ] **Step 1: `query_orders` tool**

```ts
{
  name: 'query_orders',
  description: 'Search orders by status, customer, stage, date range. Returns up to 20 matching orders with summary fields.',
  inputSchema: z.object({
    status: z.enum(['active','completed','cancelled']).optional(),
    customerName: z.string().optional(),
    stageKey: z.string().optional(),
    fromDate: z.string().optional(),
    toDate: z.string().optional(),
  }),
  handler: async (args, ctx) => {
    // Apply user's team scope (Phase 2 service)
    const scope = applyOrderScope({ user: ctx.user });
    const results = await db.select(/*...*/).from(orders).where(/* combined filters + scope */).limit(20);
    return { orders: results.map(summary), totalEstimate: results.length };
  }
}
```

- [ ] **Step 2: `get_order` tool — full detail of single order including stages + items**

- [ ] **Step 3: `query_ncrs` / `query_capas` / `query_complaints` follow same shape**

- [ ] **Step 4: Each tool call audit-logged with action `mcp.{tool_name}` and detail_json containing args + result count**

- [ ] **Step 5: Tools never expose: passwords, e-sig tokens, push subscription endpoints, internal IDs only used for joins**

- [ ] **Step 6: Result truncation: text fields cap at 2000 chars; lists at 20 items; raw JSON max 10KB per response**

- [ ] **Step 7: Commit**

```bash
git add apps/mcp/src/tools/orders.ts apps/mcp/src/tools/ncrs.ts apps/mcp/src/tools/capas.ts apps/mcp/src/tools/complaints.ts
git commit -m "feat(phase4): MCP read-only tools — orders, ncrs, capas, complaints with team scope"
```

---

#### Task 20: MCP tools — documents, training, audit, reports

**Files:**
- Create: `apps/mcp/src/tools/documents.ts`
- Create: `apps/mcp/src/tools/training.ts`
- Create: `apps/mcp/src/tools/audit.ts`
- Create: `apps/mcp/src/tools/reports.ts`

- [ ] **Step 1: `find_document` — search controlled documents by code, title, status**

- [ ] **Step 2: `check_training_status` — returns own training records (or admin can query others)**

- [ ] **Step 3: `search_audit_log` — admin/qa_manager only, filters by entity_type/action/date, max 100 rows**

- [ ] **Step 4: `generate_management_summary` — produces a brief markdown summary of last 30 days (open NCRs, pending CAPAs, complaints, training overdue)**

- [ ] **Step 5: All tool calls audit-logged**

- [ ] **Step 6: Commit**

```bash
git add apps/mcp/src/tools/documents.ts apps/mcp/src/tools/training.ts apps/mcp/src/tools/audit.ts apps/mcp/src/tools/reports.ts
git commit -m "feat(phase4): MCP read-only tools — documents, training, audit, reports"
```

---

#### Task 21: MCP audit logging + rate limiting

**Files:**
- Create: `apps/mcp/src/audit.ts`
- Modify: `apps/mcp/src/index.ts`

- [ ] **Step 1: Wrapper that audits every tool invocation**

```ts
export async function withAudit<T>(toolName: string, args: any, ctx: McpContext, fn: () => Promise<T>): Promise<T> {
  const start = Date.now();
  let result: T | undefined;
  let errorMsg: string | undefined;
  try {
    result = await fn();
    return result;
  } catch (e) {
    errorMsg = (e as Error).message;
    throw e;
  } finally {
    await writeAudit(ctx.db, {
      actor_user_id: ctx.user.id,
      action: `mcp.${toolName}`,
      entity_type: 'mcp_tool_call',
      entity_id: ctx.requestId,
      detail_json: JSON.stringify({
        args: redactSecrets(args),
        durationMs: Date.now() - start,
        error: errorMsg,
        resultSummary: errorMsg ? null : summarize(result),
      }),
    });
  }
}
```

- [ ] **Step 2: Rate limit: 60 calls / minute / user via Cloudflare Rate Limiting rule**

- [ ] **Step 3: All MCP audit entries become part of the same hash chain**

- [ ] **Step 4: Commit**

```bash
git add apps/mcp/src/audit.ts apps/mcp/src/index.ts
git commit -m "feat(phase4): MCP audit logging wrapper + rate limit"
```

---

#### Task 22: Teams Copilot manifest + connector registration

**Files:**
- Create: `apps/mcp/teams/manifest.json`
- Create: `apps/mcp/teams/icon-192.png`
- Create: `apps/mcp/teams/README.md`

- [ ] **Step 1: Author Teams app manifest pointing to MCP /sse endpoint**

- [ ] **Step 2: README documents how to register the connector in Teams Admin Center + assign to user group**

- [ ] **Step 3: Confirm with internal IT before publishing**

- [ ] **Step 4: Commit**

```bash
git add apps/mcp/teams/
git commit -m "feat(phase4): Teams Copilot manifest + registration docs"
```

---

### Week 24: Verification + ops procedures

#### Task 23: PQ-4 verification scenario

**Files:**
- Create: `apps/api/src/__tests__/pq4-audit-mcp-validation.test.ts`

- [ ] **Step 1: End-to-end scenario**

1. Trigger annual audit report generation for prior calendar year
2. PDF generated, R2-uploaded, sha256 computed, audit_reports row inserted as draft
3. Admin signs off → e-sig + RFC 3161 timestamp request fires immediately
4. PDF SHA-256 verified against stored hash
5. Hourly timestamp batch runs → covers all audit_log entries since last batch
6. Verify endpoint passes for last 3 timestamp tokens
7. Tamper test: directly UPDATE audit_log row_hash via a separate sqlite connection (bypassing trigger only in test) → verify endpoint reports broken at correct seq
8. Schedule + conduct + sign-off a management review with auto-populated metrics
9. Schedule + run a DR drill (D1 corruption scenario) → record RTO/RPO actual + result + sign-off
10. Run validation OQ-003 (e-sig framework) → record result with e-sig + evidence
11. Traceability matrix regenerates and shows OQ-003 covers REQ-7-01..04
12. MCP query: ask "How many NCRs are open older than 30 days?" via MCP test client → returns numeric answer with audit entry
13. MCP attempts an unauthorized scope (write) → request rejected, audit entry shows error
14. MCP tool call appears in audit log with action `mcp.query_ncrs` and is included in next timestamp batch

- [ ] **Step 2: Run scenario as integration test**

```bash
bun test apps/api/src/__tests__/pq4-audit-mcp-validation.test.ts
```

Expected: all 14 sub-steps PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/__tests__/pq4-audit-mcp-validation.test.ts
git commit -m "test(phase4): PQ-4 audit report + MCP + validation + DR drill verification"
```

---

#### Task 24: Operations runbook

**Files:**
- Create: `packages/validation/ops/RUNBOOK-001-incident-response.md`
- Create: `packages/validation/ops/RUNBOOK-002-secret-rotation.md`
- Create: `packages/validation/ops/RUNBOOK-003-restore-from-backup.md`

- [ ] **Step 1: RUNBOOK-001 covers**

- Incident severity classification
- On-call rotation
- Escalation path (admin → qa_manager → external regulatory if required)
- Audit log preservation (never delete; never edit; export to safe location)
- Communication templates

- [ ] **Step 2: RUNBOOK-002 covers**

- JWT signing key rotation (key versioning + dual-issue grace period)
- VAPID keys for push
- Cloudflare API tokens
- Entra OIDC client secret
- TSA endpoint change (with backward compat for existing tokens)

- [ ] **Step 3: RUNBOOK-003 covers**

- D1 PITR window + procedure
- R2 immutable bucket bypass via support ticket only
- Verification checklist post-restore (audit chain valid? e-sig records intact? RFC 3161 tokens still verify?)

- [ ] **Step 4: Commit**

```bash
git add packages/validation/ops/
git commit -m "docs(phase4): operations runbooks — incident, rotation, restore"
```

---

#### Task 25: Phase 4 deploy + post-deploy validation execution

**Files:**
- Modify: `apps/api/wrangler.toml`
- Modify: `apps/mcp/wrangler.toml`

- [ ] **Step 1: Deploy api + mcp + web to staging**

- [ ] **Step 2: Execute PQ-001 / PQ-002 / PQ-003 / PQ-004 against staging**

- [ ] **Step 3: Record validation_runs entries with admin e-sig**

- [ ] **Step 4: Generate first annual audit report (test period — quarter Q1)**

- [ ] **Step 5: Run DR drill scenario 1 (D1 PITR exercise)**

- [ ] **Step 6: Sign off all results — system enters "validated" state**

- [ ] **Step 7: Promote staging → production**

- [ ] **Step 8: Commit**

```bash
git add apps/api/wrangler.toml apps/mcp/wrangler.toml
git commit -m "chore(phase4): production deploy config + post-deploy validation execution checklist"
```

---

## Phase 4 Exit Criteria

- [ ] Annual QMS audit report PDF generates with all 9 audit areas + 2 appendices
- [ ] Report sign-off flows through e-sig + R2 immutable + RFC 3161 timestamp
- [ ] Management review minutes generate with auto-populated inputs and TipTap-edited outputs
- [ ] DR drill recorded with RTO/RPO + e-sig sign-off; report PDF in R2 immutable
- [ ] VMP + 2 IQ + 5 OQ + 3 PQ documents authored and stored under `packages/validation/`
- [ ] Validation tab UI shows live status of all protocols + traceability matrix coverage
- [ ] Hourly RFC 3161 timestamp batch runs and verifies via `/api/timestamps/verify`
- [ ] MCP Worker deployed behind Cloudflare Access; OAuth + PKCE works
- [ ] All read-only MCP tools function and audit-log every call
- [ ] MCP cannot perform any write operation (verified by test attempt)
- [ ] Teams Copilot manifest registered and Copilot can query "open NCRs" successfully
- [ ] Operations runbooks committed
- [ ] PQ-4 scenario passes end-to-end
- [ ] Hash chain verification + RFC 3161 verification both pass for full session
- [ ] Production deploy completed; first validated audit run signed off

**Approximate effort:** 5–6 weeks for one full-stack engineer

---

## Out of Scope (separate projects)

- Risk Management File generator (ISO 14971) — separate project
- Design History File (DHF) module per 820.30 — separate project (current Phase 1 DHR ≠ DHF)
- ISO 27001 Statement of Applicability + ISMS audit module — separate project
- HIPAA-specific PHI handling (current scope excludes patient data — only device commerce records)
- Customer-facing portal (out of scope per single-tenant internal positioning)
- Real-time Copilot streaming write commands (deliberately disallowed — too high risk for Part 11 compliance)
- E-signature mobile flow via Authenticator app (would require enterprise PKI investment)
