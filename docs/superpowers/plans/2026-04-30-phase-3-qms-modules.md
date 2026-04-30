# OrderFlow Phase 3: QMS Modules (NCR / CAPA / Complaints / Document Control / Training) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the five core QMS modules required by 21 CFR 820 / ISO 13485 — Nonconformance Reports (820.90), Corrective and Preventive Action (820.100), Complaints handling (820.198), Quality Document Control (820.40), and Training Records (820.25). Each module integrates with the existing order/audit/e-signature infrastructure from Phase 0–2.

**Architecture:** Add five vertical slices, each with its own DB tables, routes, services, and UI feature folder. All five reuse the e-signature framework (Phase 0), hash-chain audit log, and 7-role RBAC + SoD rules. Wire NCR ↔ orders (NCR can be raised from a failed acceptance record), CAPA ↔ NCR/Complaints (CAPA derives from one or many findings), Complaints ↔ orders (complaint references shipped order/Lot), Document Control ↔ stage_templates (template versions are controlled documents), Training ↔ users (role permissions gate on completed_trainings_json which now becomes structured records).

**Tech Stack:** Same as Phase 1–2. Add: `@tiptap/react` (rich-text editor for document body and CAPA root-cause analysis), `pdfjs-dist` (preview controlled documents in-browser), `papaparse` (CSV import for legacy training records).

**Phase boundary:** Ends when (a) any user can raise an NCR from a failed acceptance record or standalone, route through MRB disposition with e-sig; (b) one or many NCRs/Complaints can be aggregated into a CAPA with root-cause analysis (5-Why or Fishbone), action plan, effectiveness check, and dual e-sig closure; (c) Complaints flow from intake → investigation → reportability decision (FDA MDR / TFDA AE) → closure with e-sig; (d) controlled documents are created, reviewed, approved (dual e-sig), released, superseded, and obsoleted with version history; (e) training records track each user's competency per role with periodic refresher reminders. Annual audit and MCP server deferred to Phase 4.

---

## Reference Spec

Cross-check tasks against `/Users/swryociao/orderflow/docs/superpowers/specs/2026-04-30-orderflow-design.md`:
- §3 Regulatory mapping (820.40, 820.90, 820.100, 820.198, 820.25)
- §4 RBAC + Separation of Duties (SoD-2 NCR reporter≠disposer, SoD-3 CAPA owner≠closer, SoD-4 doc creator≠approver)
- §7 Electronic signatures (re-auth, meaning, reason)
- §8 Audit log hash chain
- §17 ALCOA+ data integrity attributes
- §18 Document control lifecycle (Draft → In Review → Approved → Released → Superseded / Obsolete)

---

## File Structure

```
orderflow/
├── apps/api/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── ncr.ts
│   │   │   ├── capa.ts
│   │   │   ├── complaints.ts
│   │   │   ├── documents.ts
│   │   │   └── training.ts
│   │   ├── services/
│   │   │   ├── ncrService.ts
│   │   │   ├── capaService.ts
│   │   │   ├── complaintsService.ts
│   │   │   ├── documentsService.ts
│   │   │   ├── trainingService.ts
│   │   │   └── reportabilityService.ts        # FDA MDR / TFDA AE decision support
│   │   └── db/
│   │       └── migrations/
│   │           ├── 0004_ncr.sql
│   │           ├── 0005_capa.sql
│   │           ├── 0006_complaints.sql
│   │           ├── 0007_documents.sql
│   │           └── 0008_training.sql
└── apps/web/
    └── src/
        └── features/
            ├── ncr/
            │   ├── NcrListView.tsx
            │   ├── NcrDetailView.tsx
            │   ├── NcrCreateDialog.tsx
            │   ├── MrbDispositionDialog.tsx
            │   └── NcrFromAcceptanceFlow.tsx     # Wired from failed acceptance record
            ├── capa/
            │   ├── CapaListView.tsx
            │   ├── CapaDetailView.tsx
            │   ├── RootCauseEditor.tsx           # 5-Why + Fishbone
            │   ├── ActionPlanEditor.tsx
            │   ├── EffectivenessCheckDialog.tsx
            │   └── CapaCloseDialog.tsx
            ├── complaints/
            │   ├── ComplaintIntakeView.tsx
            │   ├── ComplaintDetailView.tsx
            │   ├── ReportabilityDecisionDialog.tsx
            │   └── ComplaintCloseDialog.tsx
            ├── documents/
            │   ├── DocumentLibraryView.tsx
            │   ├── DocumentEditor.tsx             # TipTap editor
            │   ├── DocumentReviewView.tsx
            │   ├── DocumentApprovalDialog.tsx
            │   ├── VersionHistoryView.tsx
            │   └── DocumentViewer.tsx             # PDF.js preview
            └── training/
                ├── TrainingMatrixView.tsx
                ├── TrainingRecordView.tsx
                ├── TrainingAssignmentDialog.tsx
                ├── CompletionDialog.tsx
                └── TrainingReminderInbox.tsx
```

---

## Task List

### Week 13: NCR module (820.90)

#### Task 1: NCR DB schema migration

**Files:**
- Create: `apps/api/src/db/migrations/0004_ncr.sql`
- Modify: `apps/api/src/db/schema.ts`

- [ ] **Step 1: Write migration SQL**

```sql
CREATE TABLE ncrs (
  id TEXT PRIMARY KEY,
  ncr_number TEXT NOT NULL UNIQUE,             -- e.g. NCR-2026-0001
  reported_by TEXT NOT NULL REFERENCES users(id),
  reported_at TIMESTAMP NOT NULL,
  source_type TEXT NOT NULL,                   -- acceptance_record | order | external_audit | internal_audit | self_report
  source_id TEXT,                              -- references the source record
  related_order_id TEXT REFERENCES orders(id),
  related_item_id TEXT REFERENCES order_items(id),
  related_lot TEXT,
  related_gtin TEXT,
  description TEXT NOT NULL,
  severity TEXT NOT NULL,                      -- minor | major | critical
  category TEXT NOT NULL,                      -- material | process | equipment | documentation | other
  immediate_containment TEXT,                  -- what was done now to limit impact
  status TEXT NOT NULL DEFAULT 'open',         -- open | in_review | disposition_decided | closed | escalated_to_capa
  mrb_decision TEXT,                           -- accept_use_as_is | rework | repair | scrap | return_to_supplier | hold_pending_capa
  mrb_decision_by TEXT REFERENCES users(id),
  mrb_decision_at TIMESTAMP,
  mrb_disposition_esig_id TEXT REFERENCES e_signatures(id),
  mrb_justification TEXT,
  closure_reason TEXT,
  closed_by TEXT REFERENCES users(id),
  closed_at TIMESTAMP,
  closure_esig_id TEXT REFERENCES e_signatures(id),
  capa_id TEXT,                                -- if escalated to CAPA
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ncrs_status ON ncrs(status);
CREATE INDEX idx_ncrs_reported_by ON ncrs(reported_by);
CREATE INDEX idx_ncrs_related_order ON ncrs(related_order_id);

-- Attachments (photos, supplier reports, etc.)
CREATE TABLE ncr_attachments (
  id TEXT PRIMARY KEY,
  ncr_id TEXT NOT NULL REFERENCES ncrs(id),
  document_id TEXT NOT NULL REFERENCES documents(id),
  uploaded_by TEXT NOT NULL REFERENCES users(id),
  uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Discussion thread on NCR (free-text comments, all immutable per ALCOA+)
CREATE TABLE ncr_comments (
  id TEXT PRIMARY KEY,
  ncr_id TEXT NOT NULL REFERENCES ncrs(id),
  author_id TEXT NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER no_ncr_comments_update BEFORE UPDATE ON ncr_comments
BEGIN SELECT RAISE(ABORT, 'comments are immutable per ALCOA+'); END;
CREATE TRIGGER no_ncr_comments_delete BEFORE DELETE ON ncr_comments
BEGIN SELECT RAISE(ABORT, 'comments are immutable per ALCOA+'); END;
```

- [ ] **Step 2: Update Drizzle schema**

- [ ] **Step 3: Apply migration to local D1**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/db/migrations/0004_ncr.sql apps/api/src/db/schema.ts
git commit -m "feat(phase3): NCR schema (820.90) — ncrs, attachments, immutable comments"
```

---

#### Task 2: NCR number generator + service core

**Files:**
- Create: `apps/api/src/services/ncrService.ts`
- Create: `apps/api/src/services/__tests__/ncrService.test.ts`

- [ ] **Step 1: Write failing tests**

```ts
test('createNcr generates NCR-YYYY-NNNN number', async () => {
  const ncr = await createNcr({
    reportedBy: 'u1', sourceType: 'acceptance_record', sourceId: 'acc_1',
    description: 'Lot 1234 dimension out of spec',
    severity: 'major', category: 'material',
  });
  expect(ncr.ncrNumber).toMatch(/^NCR-2026-\d{4}$/);
});

test('createNcr from acceptance record copies Lot/GTIN automatically', async () => {
  const acc = await seedAcceptance({ result: 'fail', orderItemId: 'item_1', lot: 'L123', gtin: 'G456' });
  const ncr = await createNcrFromAcceptance({ reportedBy: 'u1', acceptanceId: acc.id });
  expect(ncr.relatedLot).toBe('L123');
  expect(ncr.relatedGtin).toBe('G456');
  expect(ncr.severity).toBe('major');
});

test('SoD-2: same user cannot raise + disposition same NCR', async () => {
  const ncr = await createNcr({ reportedBy: 'u1', /* ... */ });
  await expect(decideDisposition({ ncrId: ncr.id, deciderId: 'u1', /* ... */ }))
    .rejects.toThrow(/SoD-2/);
});
```

- [ ] **Step 2: Implement createNcr + createNcrFromAcceptance**

```ts
export async function createNcr(input: CreateNcrInput) {
  return db.transaction(async (tx) => {
    const year = new Date().getFullYear();
    const seq = await nextNcrSeq(tx, year);
    const ncrNumber = `NCR-${year}-${seq.toString().padStart(4, '0')}`;
    const id = ulid();
    await tx.insert(ncrs).values({
      id, ncrNumber, reportedBy: input.reportedBy, reportedAt: new Date(),
      sourceType: input.sourceType, sourceId: input.sourceId,
      relatedOrderId: input.relatedOrderId, relatedItemId: input.relatedItemId,
      relatedLot: input.relatedLot, relatedGtin: input.relatedGtin,
      description: input.description, severity: input.severity, category: input.category,
      immediateContainment: input.immediateContainment, status: 'open',
    });
    await writeAudit(tx, {
      actor_user_id: input.reportedBy,
      action: 'ncr.create',
      entity_type: 'ncr', entity_id: id,
      detail_json: JSON.stringify({ ncrNumber, severity: input.severity }),
    });
    return { id, ncrNumber };
  });
}

async function nextNcrSeq(tx: any, year: number): Promise<number> {
  const last = await tx.select({ max: sql`MAX(ncr_number)` }).from(ncrs)
    .where(like(ncrs.ncrNumber, `NCR-${year}-%`));
  if (!last[0].max) return 1;
  return Number((last[0].max as string).split('-')[2]) + 1;
}

export async function createNcrFromAcceptance({ reportedBy, acceptanceId }) {
  const acc = await db.query.acceptanceRecords.findFirst({ where: eq(acceptanceRecords.id, acceptanceId) });
  if (!acc) throw new Error('acceptance not found');
  if (acc.result !== 'fail') throw new Error('only failed acceptances can spawn NCR');
  const item = await db.query.orderItems.findFirst({ where: eq(orderItems.id, acc.orderItemId) });
  return createNcr({
    reportedBy,
    sourceType: 'acceptance_record', sourceId: acc.id,
    relatedOrderId: item?.orderId, relatedItemId: item?.id,
    relatedLot: item?.lot, relatedGtin: item?.gtin,
    description: `Failed ${acc.stage} inspection. Method: ${acc.method}. Notes: ${acc.notes ?? ''}`,
    severity: 'major', category: 'material',
  });
}
```

- [ ] **Step 3: Implement decideDisposition with SoD-2 enforcement**

```ts
export async function decideDisposition({ ncrId, deciderId, decision, justification, esigToken, esigPassword, reason }) {
  return db.transaction(async (tx) => {
    const ncr = await tx.query.ncrs.findFirst({ where: eq(ncrs.id, ncrId) });
    if (!ncr) throw new Error('not found');
    if (ncr.status === 'closed' || ncr.status === 'escalated_to_capa') throw new Error('ncr already finalized');
    if (ncr.reportedBy === deciderId) throw new Error('SoD-2: NCR reporter cannot disposition own NCR');

    const recordHash = sha256(canonicalJson({ ncrId, decision, justification, ts: new Date().toISOString() }));
    const esig = await createESignature(tx, {
      userId: deciderId, recordType: 'ncr', recordId: ncrId,
      meaning: 'approval', reason,
      recordHash, reauthToken: esigToken, password: esigPassword,
    });

    await tx.update(ncrs).set({
      mrbDecision: decision, mrbDecisionBy: deciderId, mrbDecisionAt: new Date(),
      mrbDispositionEsigId: esig.id, mrbJustification: justification,
      status: 'disposition_decided',
    }).where(eq(ncrs.id, ncrId));

    await writeAudit(tx, {
      actor_user_id: deciderId, action: 'ncr.disposition',
      entity_type: 'ncr', entity_id: ncrId,
      esig_id: esig.id,
      detail_json: JSON.stringify({ decision, justification }),
    });
  });
}
```

- [ ] **Step 4: Verify all tests pass**

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/services/ncrService.ts apps/api/src/services/__tests__/ncrService.test.ts
git commit -m "feat(phase3): ncrService — create + disposition with SoD-2 + e-sig"
```

---

#### Task 3: NCR routes + RBAC

**Files:**
- Create: `apps/api/src/routes/ncr.ts`
- Modify: `apps/api/src/index.ts`

- [ ] **Step 1: Routes**

| Method | Path | Roles |
|--------|------|-------|
| GET | /api/ncrs | admin, qa_manager, ops, sales_manager (own team), accountant (read $ impact only), viewer |
| GET | /api/ncrs/:id | same as list |
| POST | /api/ncrs | admin, qa_manager, ops |
| POST | /api/ncrs/from-acceptance | admin, qa_manager, ops |
| POST | /api/ncrs/:id/disposition | admin, qa_manager (with SoD-2) |
| POST | /api/ncrs/:id/escalate-to-capa | admin, qa_manager |
| POST | /api/ncrs/:id/close | admin, qa_manager (with SoD-2 + e-sig) |
| POST | /api/ncrs/:id/comments | any role with read access |

- [ ] **Step 2: All write actions audit-logged with esig_id when applicable**

- [ ] **Step 3: Integration test for full lifecycle**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/routes/ncr.ts apps/api/src/index.ts
git commit -m "feat(phase3): NCR REST routes with role-gated mutations"
```

---

#### Task 4: NCR list + detail UI

**Files:**
- Create: `apps/web/src/features/ncr/NcrListView.tsx`
- Create: `apps/web/src/features/ncr/NcrDetailView.tsx`
- Create: `apps/web/src/features/ncr/NcrCreateDialog.tsx`

- [ ] **Step 1: List view: filters by status / severity / reporter; defaults to "open + in_review" for QA users**

- [ ] **Step 2: Detail view shows: header, source link (clickable to acceptance / order / item), description, severity, immediate containment, comments thread, disposition section (gated by status), attachments, e-signatures rendered with manifestation block (per Part 11 11.50)**

- [ ] **Step 3: Create dialog: when source = acceptance_record, auto-populate fields; otherwise full form**

- [ ] **Step 4: Manual smoke test**

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/features/ncr/NcrListView.tsx apps/web/src/features/ncr/NcrDetailView.tsx apps/web/src/features/ncr/NcrCreateDialog.tsx
git commit -m "feat(phase3): NCR list + detail + create UI"
```

---

#### Task 5: MRB disposition dialog + NCR close flow

**Files:**
- Create: `apps/web/src/features/ncr/MrbDispositionDialog.tsx`
- Create: `apps/web/src/features/ncr/NcrFromAcceptanceFlow.tsx`

- [ ] **Step 1: MRB dialog: dropdown for decision, justification text area, e-sig dialog (re-auth + reason)**

- [ ] **Step 2: From-acceptance entry point: button on failed acceptance record cards in OrderDetailView**

- [ ] **Step 3: Close flow: requires disposition done + outcome (effectiveness verified) + e-sig**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/ncr/MrbDispositionDialog.tsx apps/web/src/features/ncr/NcrFromAcceptanceFlow.tsx
git commit -m "feat(phase3): MRB disposition dialog + raise-NCR-from-acceptance flow"
```

---

### Week 14: CAPA module (820.100)

#### Task 6: CAPA DB schema migration

**Files:**
- Create: `apps/api/src/db/migrations/0005_capa.sql`

- [ ] **Step 1: Write migration**

```sql
CREATE TABLE capas (
  id TEXT PRIMARY KEY,
  capa_number TEXT NOT NULL UNIQUE,           -- CAPA-2026-0001
  type TEXT NOT NULL,                          -- corrective | preventive
  owner_id TEXT NOT NULL REFERENCES users(id),
  qa_approver_id TEXT REFERENCES users(id),
  initiated_by TEXT NOT NULL REFERENCES users(id),
  initiated_at TIMESTAMP NOT NULL,
  trigger_summary TEXT NOT NULL,
  problem_statement TEXT NOT NULL,
  scope TEXT NOT NULL,
  root_cause_method TEXT,                      -- 5why | fishbone | other
  root_cause_analysis TEXT,                    -- TipTap HTML
  root_cause_summary TEXT,
  action_plan TEXT,                            -- TipTap HTML
  target_completion_date DATE,
  effectiveness_check_method TEXT,
  effectiveness_check_result TEXT,
  effectiveness_check_at TIMESTAMP,
  effectiveness_check_by TEXT REFERENCES users(id),
  effectiveness_esig_id TEXT REFERENCES e_signatures(id),
  status TEXT NOT NULL DEFAULT 'opened',       -- opened | analysis | action | verification | closed | cancelled
  closed_at TIMESTAMP,
  closed_by TEXT REFERENCES users(id),
  closure_esig_id TEXT REFERENCES e_signatures(id),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_capas_status ON capas(status);
CREATE INDEX idx_capas_owner ON capas(owner_id);

-- Many-to-many: NCRs / Complaints aggregated into one CAPA
CREATE TABLE capa_findings (
  id TEXT PRIMARY KEY,
  capa_id TEXT NOT NULL REFERENCES capas(id),
  finding_type TEXT NOT NULL,                  -- ncr | complaint | audit_finding | trend
  finding_id TEXT NOT NULL,                    -- references ncr.id / complaint.id / audit_finding.id
  added_by TEXT NOT NULL REFERENCES users(id),
  added_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(capa_id, finding_type, finding_id)
);
CREATE INDEX idx_capa_findings_capa ON capa_findings(capa_id);

-- Action items derived from action plan (each can be assigned + tracked)
CREATE TABLE capa_actions (
  id TEXT PRIMARY KEY,
  capa_id TEXT NOT NULL REFERENCES capas(id),
  description TEXT NOT NULL,
  assignee_id TEXT NOT NULL REFERENCES users(id),
  target_date DATE NOT NULL,
  completed_at TIMESTAMP,
  completion_evidence TEXT,
  status TEXT NOT NULL DEFAULT 'pending',      -- pending | in_progress | done | overdue
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_capa_actions_capa ON capa_actions(capa_id);
```

- [ ] **Step 2: Apply + commit**

```bash
git add apps/api/src/db/migrations/0005_capa.sql apps/api/src/db/schema.ts
git commit -m "feat(phase3): CAPA schema (820.100) — capas, findings, actions"
```

---

#### Task 7: CAPA service core (open + escalate from NCR + SoD-3)

**Files:**
- Create: `apps/api/src/services/capaService.ts`
- Create: `apps/api/src/services/__tests__/capaService.test.ts`

- [ ] **Step 1: Write failing tests**

```ts
test('escalating an NCR creates CAPA with that NCR as a finding', async () => {
  const ncr = await seedNcr({ reportedBy: 'qa1' });
  const capa = await escalateNcrToCapa({ ncrId: ncr.id, ownerId: 'eng1', initiatedBy: 'qa1' });
  expect(capa.capaNumber).toMatch(/CAPA-2026-/);
  const findings = await listFindings(capa.id);
  expect(findings).toHaveLength(1);
  expect(findings[0].findingType).toBe('ncr');
  expect(findings[0].findingId).toBe(ncr.id);
});

test('SoD-3: CAPA owner cannot close own CAPA', async () => {
  const capa = await seedCapa({ ownerId: 'eng1' });
  await expect(closeCapa({ capaId: capa.id, closedBy: 'eng1', /* ... */ }))
    .rejects.toThrow(/SoD-3/);
});

test('cannot close CAPA without effectiveness check', async () => {
  const capa = await seedCapa({ status: 'action' });
  await expect(closeCapa({ capaId: capa.id, closedBy: 'qa1', /* ... */ }))
    .rejects.toThrow(/effectiveness check required/);
});
```

- [ ] **Step 2: Implement service**

```ts
export async function escalateNcrToCapa({ ncrId, ownerId, initiatedBy, ...input }) {
  return db.transaction(async (tx) => {
    const ncr = await tx.query.ncrs.findFirst({ where: eq(ncrs.id, ncrId) });
    if (!ncr) throw new Error('ncr not found');
    if (ncr.status === 'escalated_to_capa') throw new Error('already escalated');

    const year = new Date().getFullYear();
    const seq = await nextCapaSeq(tx, year);
    const capaNumber = `CAPA-${year}-${seq.toString().padStart(4, '0')}`;
    const id = ulid();

    await tx.insert(capas).values({
      id, capaNumber, type: 'corrective',
      ownerId, initiatedBy, initiatedAt: new Date(),
      triggerSummary: `Escalated from ${ncr.ncrNumber}`,
      problemStatement: ncr.description,
      scope: input.scope ?? 'TBD',
      status: 'opened',
    });

    await tx.insert(capaFindings).values({
      id: ulid(), capaId: id,
      findingType: 'ncr', findingId: ncrId,
      addedBy: initiatedBy,
    });

    await tx.update(ncrs).set({ status: 'escalated_to_capa', capaId: id }).where(eq(ncrs.id, ncrId));

    await writeAudit(tx, { actor_user_id: initiatedBy, action: 'capa.escalate', entity_type: 'capa', entity_id: id, detail_json: JSON.stringify({ ncrId }) });
    return { id, capaNumber };
  });
}

export async function closeCapa({ capaId, closedBy, qaApproverId, esigToken, esigPassword, reason }) {
  return db.transaction(async (tx) => {
    const capa = await tx.query.capas.findFirst({ where: eq(capas.id, capaId) });
    if (!capa) throw new Error('not found');
    if (capa.ownerId === closedBy) throw new Error('SoD-3: CAPA owner cannot close own CAPA');
    if (!capa.effectivenessEsigId) throw new Error('effectiveness check required before close');

    const allActions = await tx.select().from(capaActions).where(eq(capaActions.capaId, capaId));
    const incomplete = allActions.filter(a => a.status !== 'done');
    if (incomplete.length > 0) throw new Error(`${incomplete.length} actions still incomplete`);

    const recordHash = sha256(canonicalJson({ capaId, capaNumber: capa.capaNumber, ts: new Date().toISOString() }));
    const esig = await createESignature(tx, {
      userId: closedBy, recordType: 'capa', recordId: capaId,
      meaning: 'approval', reason, recordHash,
      reauthToken: esigToken, password: esigPassword,
    });

    await tx.update(capas).set({
      status: 'closed', closedAt: new Date(), closedBy, closureEsigId: esig.id, qaApproverId,
    }).where(eq(capas.id, capaId));

    await writeAudit(tx, { actor_user_id: closedBy, action: 'capa.close', entity_type: 'capa', entity_id: capaId, esig_id: esig.id });
  });
}
```

- [ ] **Step 3: Verify tests pass**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/capaService.ts apps/api/src/services/__tests__/capaService.test.ts
git commit -m "feat(phase3): capaService — escalate + close with SoD-3 + effectiveness gate"
```

---

#### Task 8: CAPA effectiveness check + action items

**Files:**
- Modify: `apps/api/src/services/capaService.ts`

- [ ] **Step 1: Implement addAction / updateActionStatus / completeAction**

- [ ] **Step 2: Implement recordEffectivenessCheck (e-sig required, separate from closure e-sig)**

```ts
export async function recordEffectivenessCheck({ capaId, checkedBy, method, result, esigToken, esigPassword, reason }) {
  return db.transaction(async (tx) => {
    const capa = await tx.query.capas.findFirst({ where: eq(capas.id, capaId) });
    if (!capa) throw new Error('not found');
    if (capa.ownerId === checkedBy) throw new Error('SoD-3: CAPA owner cannot perform own effectiveness check');

    const recordHash = sha256(canonicalJson({ capaId, method, result, ts: new Date().toISOString() }));
    const esig = await createESignature(tx, {
      userId: checkedBy, recordType: 'capa_effectiveness', recordId: capaId,
      meaning: 'review', reason, recordHash,
      reauthToken: esigToken, password: esigPassword,
    });

    await tx.update(capas).set({
      effectivenessCheckMethod: method,
      effectivenessCheckResult: result,
      effectivenessCheckAt: new Date(),
      effectivenessCheckBy: checkedBy,
      effectivenessEsigId: esig.id,
      status: 'verification',
    }).where(eq(capas.id, capaId));

    await writeAudit(tx, { actor_user_id: checkedBy, action: 'capa.effectiveness_check', entity_type: 'capa', entity_id: capaId, esig_id: esig.id });
  });
}
```

- [ ] **Step 3: Tests for SoD-3 + state machine transitions**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/capaService.ts apps/api/src/services/__tests__/capaService.test.ts
git commit -m "feat(phase3): CAPA effectiveness check + action items with SoD-3"
```

---

#### Task 9: CAPA UI (list + detail + root-cause editor)

**Files:**
- Create: `apps/web/src/features/capa/CapaListView.tsx`
- Create: `apps/web/src/features/capa/CapaDetailView.tsx`
- Create: `apps/web/src/features/capa/RootCauseEditor.tsx`
- Create: `apps/web/src/features/capa/ActionPlanEditor.tsx`

- [ ] **Step 1: Install TipTap**

```bash
cd apps/web && bun add @tiptap/react @tiptap/starter-kit @tiptap/extension-link
```

- [ ] **Step 2: RootCauseEditor: tabs for "5-Why" (5 nested text inputs with auto-summary) and "Fishbone" (6M categories)**

- [ ] **Step 3: ActionPlanEditor: TipTap rich-text + structured action item table with assignee/date**

- [ ] **Step 4: List view: filter by status, type, owner; show overdue actions count per CAPA**

- [ ] **Step 5: Detail view: status timeline (opened → analysis → action → verification → closed), findings list with deep-links, action plan, effectiveness section, sig manifestations**

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/features/capa/
git commit -m "feat(phase3): CAPA list + detail + RootCauseEditor + ActionPlanEditor"
```

---

#### Task 10: CAPA effectiveness + close dialogs

**Files:**
- Create: `apps/web/src/features/capa/EffectivenessCheckDialog.tsx`
- Create: `apps/web/src/features/capa/CapaCloseDialog.tsx`

- [ ] **Step 1: EffectivenessCheckDialog: method dropdown (re-test / re-audit / monitoring period / metric improvement), result text, e-sig flow**

- [ ] **Step 2: CapaCloseDialog: requires QA approver selection (must differ from owner), final reason text, e-sig flow**

- [ ] **Step 3: Manual smoke test full lifecycle**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/capa/EffectivenessCheckDialog.tsx apps/web/src/features/capa/CapaCloseDialog.tsx
git commit -m "feat(phase3): CAPA effectiveness + close dialogs with e-sig"
```

---

### Week 15: Complaints module (820.198)

#### Task 11: Complaints DB schema migration

**Files:**
- Create: `apps/api/src/db/migrations/0006_complaints.sql`

- [ ] **Step 1: Write migration**

```sql
CREATE TABLE complaints (
  id TEXT PRIMARY KEY,
  complaint_number TEXT NOT NULL UNIQUE,        -- CMP-2026-0001
  received_at TIMESTAMP NOT NULL,
  received_by TEXT NOT NULL REFERENCES users(id),
  channel TEXT NOT NULL,                         -- email | phone | sales_rep | distributor | regulatory_authority | other
  reporter_name TEXT,
  reporter_email TEXT,
  reporter_phone TEXT,
  reporter_organization TEXT,
  related_order_id TEXT REFERENCES orders(id),
  related_lot TEXT,
  related_serial TEXT,
  related_gtin TEXT,
  device_use_context TEXT,                       -- e.g. clinical setting, home use
  description TEXT NOT NULL,
  severity TEXT NOT NULL,                        -- minor | moderate | serious | death | malfunction
  patient_harm TEXT,                             -- none | injury | hospitalization | permanent | death
  investigation_status TEXT NOT NULL DEFAULT 'received', -- received | in_investigation | reportability_decided | closed
  investigation_summary TEXT,
  investigation_at TIMESTAMP,
  investigation_by TEXT REFERENCES users(id),
  reportability_decision TEXT,                   -- not_reportable | fda_mdr_30day | fda_mdr_5day | tfda_ae | both
  reportability_decision_at TIMESTAMP,
  reportability_decision_by TEXT REFERENCES users(id),
  reportability_esig_id TEXT REFERENCES e_signatures(id),
  reportability_justification TEXT,
  fda_mdr_filed_at TIMESTAMP,
  fda_mdr_id TEXT,                               -- assigned by FDA
  tfda_ae_filed_at TIMESTAMP,
  tfda_ae_id TEXT,
  capa_id TEXT,                                  -- if elevated to CAPA
  closed_at TIMESTAMP,
  closed_by TEXT REFERENCES users(id),
  closure_reason TEXT,
  closure_esig_id TEXT REFERENCES e_signatures(id),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_complaints_status ON complaints(investigation_status);
CREATE INDEX idx_complaints_severity ON complaints(severity);
CREATE INDEX idx_complaints_lot ON complaints(related_lot);

CREATE TABLE complaint_attachments (
  id TEXT PRIMARY KEY,
  complaint_id TEXT NOT NULL REFERENCES complaints(id),
  document_id TEXT NOT NULL REFERENCES documents(id),
  uploaded_by TEXT NOT NULL REFERENCES users(id),
  uploaded_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE complaint_comments (
  id TEXT PRIMARY KEY,
  complaint_id TEXT NOT NULL REFERENCES complaints(id),
  author_id TEXT NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TRIGGER no_complaint_comments_update BEFORE UPDATE ON complaint_comments
BEGIN SELECT RAISE(ABORT, 'comments are immutable per ALCOA+'); END;
CREATE TRIGGER no_complaint_comments_delete BEFORE DELETE ON complaint_comments
BEGIN SELECT RAISE(ABORT, 'comments are immutable per ALCOA+'); END;
```

- [ ] **Step 2: Commit**

```bash
git add apps/api/src/db/migrations/0006_complaints.sql apps/api/src/db/schema.ts
git commit -m "feat(phase3): complaints schema (820.198) — complaints, attachments, immutable comments"
```

---

#### Task 12: Complaints intake + investigation + reportability service

**Files:**
- Create: `apps/api/src/services/complaintsService.ts`
- Create: `apps/api/src/services/reportabilityService.ts`

- [ ] **Step 1: Implement intakeComplaint (with auto-link to order via Lot/Serial lookup)**

```ts
export async function intakeComplaint(input: ComplaintIntakeInput) {
  return db.transaction(async (tx) => {
    let relatedOrderId: string | null = null;
    if (input.relatedLot) {
      const item = await tx.query.orderItems.findFirst({ where: eq(orderItems.lot, input.relatedLot) });
      relatedOrderId = item?.orderId ?? null;
    }
    const year = new Date().getFullYear();
    const seq = await nextComplaintSeq(tx, year);
    const complaintNumber = `CMP-${year}-${seq.toString().padStart(4, '0')}`;
    const id = ulid();
    await tx.insert(complaints).values({
      id, complaintNumber, receivedAt: new Date(), receivedBy: input.receivedBy,
      channel: input.channel, reporterName: input.reporterName,
      reporterEmail: input.reporterEmail, reporterPhone: input.reporterPhone,
      relatedOrderId, relatedLot: input.relatedLot, relatedSerial: input.relatedSerial,
      relatedGtin: input.relatedGtin, description: input.description,
      severity: input.severity, patientHarm: input.patientHarm,
      investigationStatus: 'received',
    });
    await writeAudit(tx, { actor_user_id: input.receivedBy, action: 'complaint.intake', entity_type: 'complaint', entity_id: id });
    return { id, complaintNumber };
  });
}
```

- [ ] **Step 2: Implement reportabilityService**

```ts
// FDA MDR rules (21 CFR 803):
// - 5-day for events that necessitate remedial action to prevent unreasonable risk
// - 30-day for deaths, serious injuries, malfunctions that could lead to death/serious injury if recurred
// TFDA AE: 7-day for serious AE; 30-day for all others
export interface ReportabilityRecommendation {
  decision: 'not_reportable' | 'fda_mdr_30day' | 'fda_mdr_5day' | 'tfda_ae' | 'both';
  rationale: string;
  fda_due_date?: string;
  tfda_due_date?: string;
}

export function recommendReportability(severity: string, patientHarm: string, channel: string): ReportabilityRecommendation {
  if (patientHarm === 'death' || patientHarm === 'permanent') {
    return {
      decision: 'both',
      rationale: 'Death or permanent injury — both FDA MDR (30-day) and TFDA AE (7-day) required',
      fda_due_date: addDays(30), tfda_due_date: addDays(7),
    };
  }
  if (patientHarm === 'hospitalization' || patientHarm === 'injury') {
    return {
      decision: 'both',
      rationale: 'Serious injury — FDA MDR (30-day) and TFDA AE (7-day) likely required',
      fda_due_date: addDays(30), tfda_due_date: addDays(7),
    };
  }
  if (severity === 'malfunction') {
    return {
      decision: 'fda_mdr_30day',
      rationale: 'Malfunction that could cause death/serious injury if it recurred',
      fda_due_date: addDays(30),
    };
  }
  return { decision: 'not_reportable', rationale: 'No patient harm and no reportable malfunction' };
}
```

- [ ] **Step 3: decideReportability (e-sig required, qa_manager only)**

- [ ] **Step 4: closeComplaint with e-sig + audit**

- [ ] **Step 5: Tests**

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/services/complaintsService.ts apps/api/src/services/reportabilityService.ts
git commit -m "feat(phase3): complaints service + FDA/TFDA reportability decision support"
```

---

#### Task 13: Complaints routes

**Files:**
- Create: `apps/api/src/routes/complaints.ts`

- [ ] **Step 1: Routes**

| Method | Path | Roles |
|--------|------|-------|
| GET | /api/complaints | admin, qa_manager, sales_manager (own customers only), viewer |
| POST | /api/complaints | admin, qa_manager, sales_manager, sales (intake), accountant |
| POST | /api/complaints/:id/investigation | admin, qa_manager |
| POST | /api/complaints/:id/reportability | admin, qa_manager (with e-sig) |
| POST | /api/complaints/:id/file-fda-mdr | qa_manager (records filing) |
| POST | /api/complaints/:id/file-tfda-ae | qa_manager |
| POST | /api/complaints/:id/escalate-to-capa | admin, qa_manager |
| POST | /api/complaints/:id/close | qa_manager (with e-sig) |

- [ ] **Step 2: Reportability decision triggers immediate alert (level 4 — regulatory clock running)**

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/routes/complaints.ts
git commit -m "feat(phase3): complaints REST routes with reportability + FDA/TFDA filing"
```

---

#### Task 14: Complaints UI

**Files:**
- Create: `apps/web/src/features/complaints/ComplaintIntakeView.tsx`
- Create: `apps/web/src/features/complaints/ComplaintDetailView.tsx`
- Create: `apps/web/src/features/complaints/ReportabilityDecisionDialog.tsx`
- Create: `apps/web/src/features/complaints/ComplaintCloseDialog.tsx`

- [ ] **Step 1: Intake view: form with auto-Lot lookup, suggests recent orders for that customer**

- [ ] **Step 2: Detail view: timeline with regulatory clock countdown banner if reportable**

- [ ] **Step 3: ReportabilityDecisionDialog: shows recommendation from service, justification text, e-sig flow**

- [ ] **Step 4: Manual smoke test**

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/features/complaints/
git commit -m "feat(phase3): complaints UI — intake + detail + reportability decision"
```

---

### Week 16: Document Control module (820.40)

#### Task 15: Documents DB schema + R2 immutable mirror

**Files:**
- Create: `apps/api/src/db/migrations/0007_documents.sql`

- [ ] **Step 1: Schema covers: SOPs, Work Instructions, Forms, Specs, Drawings — all version-controlled**

```sql
-- Note: documents table created in Phase 1 for ad-hoc attachments. Now extended for controlled docs.
ALTER TABLE documents ADD COLUMN doc_type TEXT;                     -- attachment | sop | wi | form | spec | drawing | record_template
ALTER TABLE documents ADD COLUMN doc_number TEXT;                    -- e.g. SOP-PROD-001
ALTER TABLE documents ADD COLUMN title TEXT;
ALTER TABLE documents ADD COLUMN current_version_id TEXT;
ALTER TABLE documents ADD COLUMN status TEXT NOT NULL DEFAULT 'attachment'; -- attachment | draft | in_review | approved | released | superseded | obsolete
ALTER TABLE documents ADD COLUMN owner_user_id TEXT REFERENCES users(id);
ALTER TABLE documents ADD COLUMN audience_role_json TEXT;            -- JSON array of roles required to ack
CREATE UNIQUE INDEX idx_documents_doc_number ON documents(doc_number) WHERE doc_number IS NOT NULL;

CREATE TABLE document_versions (
  id TEXT PRIMARY KEY,
  document_id TEXT NOT NULL REFERENCES documents(id),
  version_label TEXT NOT NULL,                  -- v1.0, v1.1, v2.0
  body_html TEXT,                               -- TipTap output (for sop/wi)
  body_pdf_r2_key TEXT,                         -- R2 immutable key for pdf-only docs (drawings)
  change_summary TEXT NOT NULL,
  created_by TEXT NOT NULL REFERENCES users(id),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  reviewed_by TEXT REFERENCES users(id),
  reviewed_at TIMESTAMP,
  review_esig_id TEXT REFERENCES e_signatures(id),
  approved_by TEXT REFERENCES users(id),
  approved_at TIMESTAMP,
  approval_esig_id TEXT REFERENCES e_signatures(id),
  released_at TIMESTAMP,
  effective_date DATE,
  superseded_by_version_id TEXT REFERENCES document_versions(id),
  obsoleted_at TIMESTAMP,
  status TEXT NOT NULL DEFAULT 'draft',         -- draft | in_review | approved | released | superseded | obsolete
  UNIQUE(document_id, version_label)
);
CREATE INDEX idx_doc_versions_doc ON document_versions(document_id);

-- Acknowledgements: users who must read released doc
CREATE TABLE document_acknowledgements (
  id TEXT PRIMARY KEY,
  version_id TEXT NOT NULL REFERENCES document_versions(id),
  user_id TEXT NOT NULL REFERENCES users(id),
  acknowledged_at TIMESTAMP,
  ack_esig_id TEXT REFERENCES e_signatures(id),
  due_date DATE,
  UNIQUE(version_id, user_id)
);
CREATE INDEX idx_doc_ack_user ON document_acknowledgements(user_id);
```

- [ ] **Step 2: Apply + commit**

```bash
git add apps/api/src/db/migrations/0007_documents.sql apps/api/src/db/schema.ts
git commit -m "feat(phase3): document control schema (820.40) — versions, acks, immutable PDFs"
```

---

#### Task 16: Document service — lifecycle state machine

**Files:**
- Create: `apps/api/src/services/documentsService.ts`
- Create: `apps/api/src/services/__tests__/documentsService.test.ts`

- [ ] **Step 1: State machine**

```
draft → in_review (by owner) → approved (by admin/qa_manager, dual-sign) → released (after effective_date) → superseded (when newer version released) → obsolete (manual or automatic when superseded older than 7 years)
```

- [ ] **Step 2: SoD-4 enforcement: creator cannot review or approve own version**

```ts
test('SoD-4: creator cannot review own version', async () => {
  const v = await createVersion({ documentId, body, createdBy: 'u1' });
  await expect(submitReview({ versionId: v.id, reviewedBy: 'u1', /*...*/ }))
    .rejects.toThrow(/SoD-4/);
});

test('SoD-4: reviewer cannot approve same version', async () => {
  const v = await createVersion({ createdBy: 'u1' });
  await submitReview({ versionId: v.id, reviewedBy: 'u2', /*...*/ });
  await expect(approveVersion({ versionId: v.id, approvedBy: 'u2', /*...*/ }))
    .rejects.toThrow(/SoD-4/);
});
```

- [ ] **Step 3: Approval requires re-auth e-sig + reason; release auto-supersedes prior released version**

- [ ] **Step 4: When version released, auto-create document_acknowledgements rows for all users matching the doc's audience_role_json**

- [ ] **Step 5: Implement service + tests**

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/services/documentsService.ts apps/api/src/services/__tests__/documentsService.test.ts
git commit -m "feat(phase3): documentsService — lifecycle + SoD-4 + auto ack on release"
```

---

#### Task 17: Document routes + R2 immutable storage

**Files:**
- Create: `apps/api/src/routes/documents.ts`

- [ ] **Step 1: Routes**

| Method | Path | Roles |
|--------|------|-------|
| GET | /api/documents | all (released only for non-admin) |
| GET | /api/documents/:id | all (released or own draft) |
| POST | /api/documents | admin, qa_manager (define new SOP) |
| POST | /api/documents/:id/versions | admin, qa_manager, ops, sales_manager (create draft) |
| POST | /api/documents/:id/versions/:vid/submit-review | owner |
| POST | /api/documents/:id/versions/:vid/review | reviewer (with SoD-4 + e-sig) |
| POST | /api/documents/:id/versions/:vid/approve | approver (with SoD-4 + e-sig) |
| POST | /api/documents/:id/versions/:vid/release | admin |
| POST | /api/documents/:id/versions/:vid/obsolete | admin |
| POST | /api/documents/:id/acknowledge | logged-in user (with e-sig) |

- [ ] **Step 2: When version becomes "approved", body PDF uploaded to R2-immutable bucket with key `documents/{docId}/{versionLabel}.pdf`; bucket lifecycle policy = no delete for 7 years**

- [ ] **Step 3: Tests**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/routes/documents.ts
git commit -m "feat(phase3): documents routes + R2 immutable storage on approval"
```

---

#### Task 18: Document UI — library + editor + viewer

**Files:**
- Create: `apps/web/src/features/documents/DocumentLibraryView.tsx`
- Create: `apps/web/src/features/documents/DocumentEditor.tsx`
- Create: `apps/web/src/features/documents/DocumentViewer.tsx`
- Create: `apps/web/src/features/documents/VersionHistoryView.tsx`

- [ ] **Step 1: Install pdfjs**

```bash
cd apps/web && bun add pdfjs-dist
```

- [ ] **Step 2: DocumentLibraryView: table of all released docs with doc_number, title, current version, effective date; filter by doc_type**

- [ ] **Step 3: DocumentEditor: TipTap rich-text for body_html OR file upload for PDF-only types; "Save as Draft" / "Submit for Review"**

- [ ] **Step 4: DocumentViewer: renders body_html or PDF (using pdfjs); shows version label + status badge; "Acknowledge" CTA if user has open ack with e-sig flow**

- [ ] **Step 5: VersionHistoryView: ordered list of all versions with status, who created/reviewed/approved, e-sig manifestation block, supersedes/superseded-by links**

- [ ] **Step 6: Commit**

```bash
git add apps/web/src/features/documents/
git commit -m "feat(phase3): document library + editor + viewer + version history"
```

---

#### Task 19: Document review + approval dialogs

**Files:**
- Create: `apps/web/src/features/documents/DocumentReviewView.tsx`
- Create: `apps/web/src/features/documents/DocumentApprovalDialog.tsx`

- [ ] **Step 1: Review view: in_review queue for users in reviewer pool with diff vs prior version**

- [ ] **Step 2: ApprovalDialog: shows full version + change summary; e-sig with reason mandatory; approver must differ from creator and reviewer (SoD-4)**

- [ ] **Step 3: Manual smoke test**

- [ ] **Step 4: Commit**

```bash
git add apps/web/src/features/documents/DocumentReviewView.tsx apps/web/src/features/documents/DocumentApprovalDialog.tsx
git commit -m "feat(phase3): document review + approval dialogs with SoD-4 + e-sig"
```

---

### Week 17: Training Records module (820.25)

#### Task 20: Training DB schema migration

**Files:**
- Create: `apps/api/src/db/migrations/0008_training.sql`

- [ ] **Step 1: Schema**

```sql
CREATE TABLE training_curricula (
  id TEXT PRIMARY KEY,
  curriculum_code TEXT NOT NULL UNIQUE,         -- TRN-OPS-001
  title TEXT NOT NULL,
  description TEXT,
  required_for_roles TEXT NOT NULL,             -- JSON array of roles
  required_for_stages TEXT,                     -- JSON array of stage_keys
  refresher_interval_months INTEGER,            -- null = no refresher required
  document_id TEXT REFERENCES documents(id),    -- linked SOP/training material
  active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE training_records (
  id TEXT PRIMARY KEY,
  curriculum_id TEXT NOT NULL REFERENCES training_curricula(id),
  user_id TEXT NOT NULL REFERENCES users(id),
  assigned_at TIMESTAMP NOT NULL,
  assigned_by TEXT NOT NULL REFERENCES users(id),
  due_date DATE,
  completed_at TIMESTAMP,
  completed_by TEXT REFERENCES users(id),
  trainer_id TEXT REFERENCES users(id),
  completion_method TEXT,                       -- self_study | classroom | otj | exam
  exam_score INTEGER,
  exam_passed BOOLEAN,
  evidence_doc_id TEXT REFERENCES documents(id),
  completion_esig_id TEXT REFERENCES e_signatures(id),
  expires_at DATE,                              -- computed = completed_at + refresher_interval
  status TEXT NOT NULL DEFAULT 'assigned',      -- assigned | in_progress | completed | expired | waived
  waiver_reason TEXT,
  waiver_esig_id TEXT REFERENCES e_signatures(id),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_training_records_user ON training_records(user_id);
CREATE INDEX idx_training_records_status ON training_records(status);
CREATE INDEX idx_training_records_expires ON training_records(expires_at);
```

- [ ] **Step 2: Commit**

```bash
git add apps/api/src/db/migrations/0008_training.sql apps/api/src/db/schema.ts
git commit -m "feat(phase3): training records schema (820.25) — curricula + records"
```

---

#### Task 21: Training service + competency gate

**Files:**
- Create: `apps/api/src/services/trainingService.ts`
- Modify: `apps/api/src/services/stageEngine.ts` (Phase 1)

- [ ] **Step 1: Implement assignTraining + completeTraining + waiveTraining**

```ts
export async function completeTraining({ recordId, completedBy, method, examScore, examPassed, evidenceDocId, esigToken, esigPassword, reason }) {
  return db.transaction(async (tx) => {
    const r = await tx.query.trainingRecords.findFirst({ where: eq(trainingRecords.id, recordId) });
    if (!r) throw new Error('not found');
    if (r.userId !== completedBy && completedBy !== r.trainerId) throw new Error('only assignee or trainer can complete');
    if (examPassed === false) throw new Error('cannot complete with failing exam — reschedule');

    const recordHash = sha256(canonicalJson({ recordId, method, examScore, ts: new Date().toISOString() }));
    const esig = await createESignature(tx, {
      userId: completedBy, recordType: 'training_record', recordId,
      meaning: 'review', reason, recordHash,
      reauthToken: esigToken, password: esigPassword,
    });

    const curriculum = await tx.query.trainingCurricula.findFirst({ where: eq(trainingCurricula.id, r.curriculumId) });
    const expiresAt = curriculum?.refresherIntervalMonths
      ? addMonths(new Date(), curriculum.refresherIntervalMonths)
      : null;

    await tx.update(trainingRecords).set({
      completedAt: new Date(), completedBy, completionMethod: method,
      examScore, examPassed, evidenceDocId, completionEsigId: esig.id,
      expiresAt, status: 'completed',
    }).where(eq(trainingRecords.id, recordId));

    await writeAudit(tx, { actor_user_id: completedBy, action: 'training.complete', entity_type: 'training_record', entity_id: recordId, esig_id: esig.id });
  });
}

// Competency gate — call before allowing user to act on a stage
export async function requireCompetency({ userId, stageKey }: { userId: string; stageKey: string }) {
  const required = await db.select().from(trainingCurricula)
    .where(sql`json_extract(required_for_stages, '$') LIKE ${`%"${stageKey}"%`} AND active = TRUE`);
  if (required.length === 0) return;

  const userRecords = await db.select().from(trainingRecords)
    .where(and(eq(trainingRecords.userId, userId), eq(trainingRecords.status, 'completed')));
  const validCurricula = new Set(
    userRecords.filter(r => !r.expiresAt || new Date(r.expiresAt) > new Date()).map(r => r.curriculumId)
  );
  const missing = required.filter(c => !validCurricula.has(c.id));
  if (missing.length > 0) {
    throw new CompetencyError(`Training required: ${missing.map(c => c.curriculumCode).join(', ')}`);
  }
}
```

- [ ] **Step 2: Wire requireCompetency into stage advance for stages with required training**

- [ ] **Step 3: Tests for assignment, completion, expiry, gate enforcement**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/trainingService.ts apps/api/src/services/stageEngine.ts apps/api/src/services/__tests__/trainingService.test.ts
git commit -m "feat(phase3): trainingService — assign/complete/waive + stage competency gate"
```

---

#### Task 22: Training routes

**Files:**
- Create: `apps/api/src/routes/training.ts`

- [ ] **Step 1: Routes**

| Method | Path | Roles |
|--------|------|-------|
| GET | /api/training/curricula | all |
| POST | /api/training/curricula | admin, qa_manager |
| GET | /api/training/records | self only OR admin/qa_manager (all) |
| GET | /api/training/records/me | self |
| POST | /api/training/records | admin, qa_manager (assign to user) |
| POST | /api/training/records/:id/complete | self OR trainer (with e-sig) |
| POST | /api/training/records/:id/waive | admin (with e-sig + reason) |
| GET | /api/training/matrix | admin, qa_manager (all users × all curricula) |

- [ ] **Step 2: Commit**

```bash
git add apps/api/src/routes/training.ts
git commit -m "feat(phase3): training REST routes"
```

---

#### Task 23: Training UI — matrix + record + reminder inbox

**Files:**
- Create: `apps/web/src/features/training/TrainingMatrixView.tsx`
- Create: `apps/web/src/features/training/TrainingRecordView.tsx`
- Create: `apps/web/src/features/training/CompletionDialog.tsx`
- Create: `apps/web/src/features/training/TrainingReminderInbox.tsx`

- [ ] **Step 1: TrainingMatrixView: heatmap of users × curricula (cells: green=valid, amber=expiring, red=missing/expired)**

- [ ] **Step 2: TrainingRecordView: per-user list of all training assignments + completion dates + expiry**

- [ ] **Step 3: CompletionDialog: method picker, exam score input (if applicable), evidence file upload, e-sig**

- [ ] **Step 4: TrainingReminderInbox: badge counter on nav for own pending/expiring training; clicking opens list with deep-links to training material**

- [ ] **Step 5: Commit**

```bash
git add apps/web/src/features/training/
git commit -m "feat(phase3): training UI — matrix, record, completion, reminder inbox"
```

---

#### Task 24: Training expiry cron + competency mismatch alert

**Files:**
- Create: `apps/api/src/cron/training-expiry.ts`
- Modify: `apps/api/wrangler.toml`

- [ ] **Step 1: Daily 06:00 cron**

```ts
export async function runTrainingExpiryCheck(env: Env) {
  const today = new Date();
  const in30Days = addDays(today, 30);

  await env.DB.prepare(`
    UPDATE training_records SET status = 'expired'
    WHERE status = 'completed' AND expires_at IS NOT NULL AND expires_at < ?
  `).bind(today.toISOString()).run();

  const expiring = await env.DB.prepare(`
    SELECT * FROM training_records WHERE status = 'completed'
      AND expires_at BETWEEN ? AND ?
  `).bind(today.toISOString(), in30Days.toISOString()).all();

  for (const r of expiring.results) {
    await createAlert({
      level: 2, type: 'training_expiring', userId: r.user_id, recordId: r.id,
      message: `Training ${r.curriculum_code} expires ${r.expires_at}`,
    });
  }
}
```

- [ ] **Step 2: Add to wrangler.toml cron**

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/cron/training-expiry.ts apps/api/wrangler.toml
git commit -m "feat(phase3): training expiry daily cron + 30-day reminder alerts"
```

---

### Week 18: Cross-module integration + verification

#### Task 25: Cross-link UI — show NCRs/CAPAs/Complaints on order detail

**Files:**
- Modify: `apps/web/src/features/orders/OrderDetailPage.tsx`

- [ ] **Step 1: Add panels: "Linked NCRs" / "Linked Complaints" / "Linked CAPAs (via NCRs)"**

- [ ] **Step 2: Each panel shows count + click-through**

- [ ] **Step 3: Commit**

```bash
git add apps/web/src/features/orders/OrderDetailPage.tsx
git commit -m "feat(phase3): cross-link NCR/CAPA/complaint panels on order detail"
```

---

#### Task 26: QA dashboard widgets — wire real data

**Files:**
- Modify: `apps/api/src/routes/qa-dashboard.ts` (Phase 2 stub)
- Modify: `apps/web/src/features/qa-dashboard/widgets/`

- [ ] **Step 1: Replace stub zeros with real counts**

```ts
{
  ncrOpen: count NCRs WHERE status IN ('open','in_review','disposition_decided'),
  ncrEscalatedThisMonth: count NCRs status='escalated_to_capa' AND escalated_at >= start_of_month,
  capaOpen: count CAPAs WHERE status NOT IN ('closed','cancelled'),
  capaOverdue: count CAPAs WHERE target_completion_date < today AND status != 'closed',
  complaintsOpen: count complaints WHERE investigation_status != 'closed',
  complaintsReportable: count complaints WHERE reportability_decision IS NOT NULL AND fda_mdr_filed_at IS NULL OR tfda_ae_filed_at IS NULL,
  trainingExpiring30d: count training_records WHERE expires_at BETWEEN today AND today+30d,
  trainingExpired: count training_records WHERE status = 'expired',
  documentsAwaitingReview: count document_versions WHERE status = 'in_review',
  documentsAwaitingAck: count user-pending document_acknowledgements,
}
```

- [ ] **Step 2: Each widget links to filtered list view**

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/routes/qa-dashboard.ts apps/web/src/features/qa-dashboard/widgets/
git commit -m "feat(phase3): qa dashboard wires real NCR/CAPA/complaint/training/doc counts"
```

---

#### Task 27: Trend analysis — repeat NCRs by Lot/Supplier (820.100(a))

**Files:**
- Create: `apps/api/src/services/trendAnalysisService.ts`
- Create: `apps/api/src/routes/trends.ts`
- Create: `apps/web/src/features/qa-dashboard/TrendAnalysisView.tsx`

- [ ] **Step 1: Run weekly aggregation:**

- NCRs grouped by category × month (last 12 months)
- NCRs grouped by related_gtin (top 10 by count)
- Complaints grouped by severity × month
- Show alert when same Lot has ≥3 NCRs OR same GTIN has spike (3× monthly avg)

- [ ] **Step 2: Service auto-suggests CAPA escalation for trend hits**

- [ ] **Step 3: UI: bar charts per dimension + alert banner**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/trendAnalysisService.ts apps/api/src/routes/trends.ts apps/web/src/features/qa-dashboard/TrendAnalysisView.tsx
git commit -m "feat(phase3): trend analysis — repeat NCRs + CAPA suggestion (820.100(a))"
```

---

#### Task 28: PQ-3 verification scenario

**Files:**
- Create: `apps/api/src/__tests__/pq3-qms-flow.test.ts`

- [ ] **Step 1: End-to-end scenario**

1. Final QC fails on Lot L-2026-001 → ops creates NCR via "Reject → NCR" CTA
2. NCR auto-populated with Lot, GTIN, Order ID
3. QA manager dispositions: "Hold pending CAPA" (e-sig)
4. SoD-2 enforced: ops who reported cannot disposition (verified by attempt that throws)
5. QA escalates NCR to CAPA → CAPA-2026-0001 created with that NCR as finding
6. CAPA owner = engineer; performs 5-Why root cause; saves analysis
7. Owner adds 3 action items (assigned to ops, qc tech, eng manager); each marked "done" with evidence
8. Different QA member performs effectiveness check (re-test method, result = passed) with e-sig
9. SoD-3 enforced: CAPA owner cannot close own CAPA
10. CAPA closed by yet another QA manager with e-sig (reason recorded)
11. NCR auto-updates status = 'closed' with link to CAPA closure
12. New SOP version drafted by ops manager (TRN-OPS-001 v1.1)
13. SoD-4 enforced: ops manager cannot review own draft
14. QA manager reviews draft (e-sig)
15. Different admin approves (e-sig); SoD-4 enforced
16. Released → all ops users get acknowledgement assigned
17. Ops user opens viewer, reads, acknowledges with e-sig
18. Training curriculum TRN-OPS-001 refresher assigned to all ops; user completes with exam score 95
19. Customer reports a malfunction on shipped Lot L-2025-099
20. Sales records complaint via intake; auto-linked to original order
21. QA investigates, decides reportability = fda_mdr_30day, e-signs decision
22. QA records FDA MDR filing with FDA ID
23. QA escalates complaint to a CAPA (separate from earlier one)
24. All 23 actions audited in audit_log with sequential seq, valid hash chain
25. QA dashboard reflects: 1 closed NCR, 1 closed CAPA, 1 open CAPA, 1 reported complaint, 1 released SOP, 1 expired-30d-from-now training

- [ ] **Step 2: Run scenario as integration test**

```bash
bun test apps/api/src/__tests__/pq3-qms-flow.test.ts
```

Expected: all sub-steps PASS.

- [ ] **Step 3: Commit**

```bash
git add apps/api/src/__tests__/pq3-qms-flow.test.ts
git commit -m "test(phase3): PQ-3 full QMS lifecycle — NCR→CAPA→Doc→Training→Complaint→MDR"
```

---

#### Task 29: Audit hash chain integrity verification

**Files:**
- Create: `apps/api/src/services/auditVerifyService.ts`
- Create: `apps/api/src/routes/admin-audit.ts`

- [ ] **Step 1: verifyAuditChain function**

```ts
export async function verifyAuditChain(): Promise<{ valid: boolean; brokenAtSeq?: number }> {
  const all = await db.select().from(auditLog).orderBy(auditLog.seq);
  let prev = GENESIS_PREV_HASH;
  for (const row of all) {
    const recomputed = await computeRowHash({ ...row, prev_hash: prev });
    if (row.rowHash !== recomputed) return { valid: false, brokenAtSeq: row.seq };
    if (row.prevHash !== prev) return { valid: false, brokenAtSeq: row.seq };
    prev = row.rowHash;
  }
  return { valid: true };
}
```

- [ ] **Step 2: Admin-only endpoint to trigger verification + return result**

- [ ] **Step 3: Test with seeded chain — force a tamper (raw sqlite UPDATE bypassing trigger via separate test connection) and verify detection**

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/services/auditVerifyService.ts apps/api/src/routes/admin-audit.ts apps/api/src/services/__tests__/auditVerify.test.ts
git commit -m "feat(phase3): audit chain verification service + admin endpoint"
```

---

#### Task 30: Phase 3 deploy + post-deploy QMS smoke

**Files:**
- Modify: `apps/api/wrangler.toml`

- [ ] **Step 1: Verify all migrations apply on staging D1**

- [ ] **Step 2: Deploy api + web**

- [ ] **Step 3: Manual smoke test on staging**

- [ ] Create a test NCR end-to-end
- [ ] Escalate to CAPA, verify SoD-3
- [ ] Create a test SOP draft, review, approve, release, ack
- [ ] Assign training, complete it
- [ ] Run audit chain verify endpoint → expect valid

- [ ] **Step 4: Commit**

```bash
git add apps/api/wrangler.toml
git commit -m "chore(phase3): wrangler bindings + post-deploy QMS smoke checklist"
```

---

## Phase 3 Exit Criteria

- [ ] NCR can be raised standalone or from failed acceptance, with auto-populated Lot/GTIN
- [ ] MRB disposition with e-sig works; SoD-2 enforced
- [ ] CAPA can be opened standalone or escalated from NCR; aggregates multiple findings
- [ ] CAPA root-cause analysis (5-Why + Fishbone) editor functional
- [ ] CAPA effectiveness check requires e-sig (different user than owner)
- [ ] CAPA closure requires e-sig (different user than owner; SoD-3)
- [ ] Complaints intake flow includes Lot/Serial auto-link
- [ ] Reportability decision UI surfaces FDA MDR + TFDA AE recommendation; e-sig records decision
- [ ] Document control supports draft → review → approve → release → supersede with full version history
- [ ] Document approval enforces SoD-4 (creator ≠ reviewer ≠ approver)
- [ ] Released documents trigger acknowledgement requirement for audience
- [ ] Training curricula link to required roles + stages
- [ ] Stage advance blocked if user lacks required training
- [ ] Training expiry cron alerts users 30 days ahead
- [ ] QA dashboard shows real NCR/CAPA/complaint/training/doc metrics
- [ ] Trend analysis flags repeat issues for CAPA suggestion
- [ ] PQ-3 scenario passes end-to-end
- [ ] Audit chain verifier returns valid for full session
- [ ] Code review pass

**Approximate effort:** 5–6 weeks for one full-stack engineer

---

## Out of Scope (deferred to Phase 4)

- Annual QMS audit report PDF generation (multi-section, multi-evidence, signed)
- Internal MCP server for Teams Copilot read-only QMS queries
- Software validation IQ/OQ/PQ formal protocols
- RFC 3161 trusted timestamping for audit + e-sig
- Disaster recovery drill protocol
- Periodic management review minutes generator (820.20)
- Risk Management File generator (ISO 14971 — separate project)
