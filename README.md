# OrderFlow

從訂單接收到出貨完成的**內部**訂單與品質管理系統，整合 ISO 13485 / 21 CFR 820 / 21 CFR Part 11 要求的生產／驗收／追溯／不符合品／CAPA／客訴流程。

- **使用對象**：本公司內部業務、生產、倉管、品保（QA/QC）、會計、主管、稽核員。**不**對外開放。
- **產業屬性**：自有設計製造（own design and manufacture）的 Class II 醫療器材製造商。
- **不在範圍內**：對外 SaaS／DHF（設計管制）／校驗管理／服務維修記錄／統計技術。

## 法規範圍

- ISO 13485:2016（國際 QMS 主軸）
- 衛福部食藥署（TFDA）醫療器材 GMP
- FDA 21 CFR Part 820（QSR — 美國輸入）
- FDA 21 CFR Part 11（電子記錄／電子簽章）
- UDI Rule 21 CFR 830（GS1 條碼追溯）
- ISO 27001 A.9.2.5（季度權限審查）

## 架構摘要

- **Monorepo**：Bun workspace，`apps/web`（React + Vite + Tailwind + shadcn）、`apps/api`（Cloudflare Workers + Hono + Drizzle + D1）、`apps/mcp`（內部 MCP server）。
- **單租戶**：無 `tenant_id`，組織配置存於單列 `organization` 表。
- **兩層認證**：(1) Cloudflare Access Zero Trust 驗 M365 + MFA → (2) 應用 JWT（HttpOnly，2h absolute / 30min idle）。
- **稽核軌跡**：SHA-256 hash chain + 嚴格遞增 `seq` + DB immutability triggers + 每小時 RFC 3161 trusted timestamping。
- **電子簽章**：Part 11 11.50 / 11.70 / 11.100 / 11.200 / 11.300 — 每次簽章重新驗證並寫入配對 audit row。
- **職責分離（SoD）**：建單者≠核可者；NCR 提報者≠處置者；CAPA 負責人≠結案人；文件建立者≠審查者≠核可者。
- **Lot/UDI/Serial 追溯**：GS1 AI 解析（AI 01 GTIN / AI 10 Lot / AI 17 Expiry / AI 21 Serial），條碼掃描整合驗收／QC／出貨。

## 文件

| 文件 | 內容 |
|------|------|
| [`docs/superpowers/specs/2026-04-30-orderflow-design.md`](docs/superpowers/specs/2026-04-30-orderflow-design.md) | 完整設計規格 v0.2（QMS / Part 11 重新定位） |
| [`docs/superpowers/plans/2026-04-30-phase-0-foundation.md`](docs/superpowers/plans/2026-04-30-phase-0-foundation.md) | Phase 0 — Foundation：M365 SSO、hash-chain audit、e-sig framework、7-role RBAC |
| [`docs/superpowers/plans/2026-04-30-phase-1-mvp.md`](docs/superpowers/plans/2026-04-30-phase-1-mvp.md) | Phase 1 — Order MVP：Lot/UDI 追溯、11 階段範本、自動 DHR、SLA、通知 |
| [`docs/superpowers/plans/2026-04-30-phase-2-mobile-advanced.md`](docs/superpowers/plans/2026-04-30-phase-2-mobile-advanced.md) | Phase 2 — Mobile PWA、UDI 條碼、進階權限、季度 access review |
| [`docs/superpowers/plans/2026-04-30-phase-3-qms-modules.md`](docs/superpowers/plans/2026-04-30-phase-3-qms-modules.md) | Phase 3 — NCR / CAPA / Complaints / Document Control / Training Records |
| [`docs/superpowers/plans/2026-04-30-phase-4-audit-mcp-validation.md`](docs/superpowers/plans/2026-04-30-phase-4-audit-mcp-validation.md) | Phase 4 — 年度稽核報表、內部 MCP（Teams Copilot）、軟體驗證 IQ/OQ/PQ、RFC 3161、DR drill |

## 實作進度

| 階段 | 週數 | 任務數 | 狀態 |
|------|------|--------|------|
| Spec v0.2 | — | — | ✅ 完成 |
| Phase 0 | 4 週 | — | ⏳ 待開工 |
| Phase 1 | 6 週 | 32 | ⏳ 規劃完成 |
| Phase 2 | 5 週 | 24 | ⏳ 規劃完成 |
| Phase 3 | 5–6 週 | 30 | ⏳ 規劃完成 |
| Phase 4 | 5–6 週 | 25 | ⏳ 規劃完成 |

預計總工期 ~25–27 週。

## 規模假設

- 並行訂單：~數十筆 ／ 月訂單量：≤ 1,000
- 員工：≤ 200
- 月 NCR：≤ 50 ／ 月客訴：≤ 30
- 法定保留期：**7 年**（醫療器材記錄）

## 開發

```bash
bun install
bun run dev
```

實際的 `apps/`、`packages/`、`migrations/` 目錄會在 Phase 0 第一個 task 建立。

## License

Proprietary — internal use only.
