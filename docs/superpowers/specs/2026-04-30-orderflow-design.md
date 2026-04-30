# OrderFlow — 業務專案進度管理系統設計規格

- 文件版本：v0.1（brainstorm 完成稿）
- 撰寫日期：2026-04-30
- 撰寫者：swlucifer@gmail.com（與 Claude 協作）
- 狀態：待 user review，後續進入 implementation plan

---

## 1. 系統定位

從訂單接收到出貨完成的業務專案進度管理系統，主要面向「製造業 + 批發/經銷」混合業態，支援多種訂單類型在同一租戶下並行運作。

- **第一階段使用對象**：本公司內部業務、生產、倉管、會計、主管。
- **長期定位**：多租戶 SaaS 架構（multi-tenant），保留外賣其他公司的能力。
- **不在範圍內**：外部客戶端介面（不開放客戶登入查詢自己訂單）。

### 1.1 核心需求

1. 同時追蹤多筆專案的執行進度（從訂單到出貨完成）。
2. 每階段預警機制，需可由管理員微調閾值。
3. 多角色權限與通知機制。
4. Web 為主、行動端 PWA 支援，含倉管條碼掃描功能。
5. 預留 Microsoft Teams MCP 整合。

### 1.2 規模假設

- 單租戶並行訂單：~數十筆
- 單租戶員工：≤ 200
- 單租戶月訂單量：≤ 5,000
- 第一年租戶數：≤ 50

---

## 2. 階段範本（Stage Templates）

提供兩套預設範本，每個租戶可微調：增刪階段、調整 SLA、開關「可選/條件式」階段。

### 2.1 製造業範本（11 階段）

| # | 階段 | 條件 / 預設啟用 | SLA 黃 / 紅 | 預設執行者 |
|---|------|---------------|------------|-----------|
| 1 | 訂單確認 | 必要 | 4h / 1d | sales |
| 2 | 客戶簽核（報價、規格確認） | 可選，預設開 | 2d / 5d | sales |
| 3 | 打款確認 — 定金 | 可選，預設開 | 3d / 7d | accountant |
| 4 | 生產排程 | 必要 | 1d / 3d | sales / ops |
| 5 | 採購備料 | 必要 | 距生產日 -2d / -0d | ops |
| 6 | 生產製造 | 必要 | 落後 20% / 40% | ops |
| 7 | 品質檢驗（QC） | 必要 | 1d / 2d | ops |
| 8 | 打款確認 — 尾款 | 可選，預設開 | 2d / 5d | accountant |
| 9 | 包裝出貨 | 必要 | 1d / 2d | ops |
| 10 | 報關（B/L、Invoice、PL） | **條件式**：`shipping_country != 'TW'` 自動啟用 | 1d / 3d | ops |
| 11 | 物流配送 | 必要 | 預計到貨 +1d / +3d | 系統自動 |

### 2.2 批發 / 經銷範本（7 階段）

| # | 階段 | 條件 | SLA 黃 / 紅 | 預設執行者 |
|---|------|------|------------|-----------|
| 1 | 訂單確認 | 必要 | 2h / 1d | sales |
| 2 | 客戶簽核 | 可選 | 1d / 3d | sales |
| 3 | 打款確認 | 可選 | 2d / 5d | accountant |
| 4 | 備貨揀貨 | 必要 | 0.5d / 2d | ops |
| 5 | 包裝出貨 | 必要 | 0.5d / 1d | ops |
| 6 | 報關 | 條件式 | 1d / 3d | ops |
| 7 | 物流配送 | 必要 | 預計到貨 +1d / +3d | 系統自動 |

### 2.3 條件式階段觸發

訂單建立或地址變更時，後端依 `shipping_country` 重新計算 active stages：
- `shipping_country == 'TW'` → 跳過「報關」
- `shipping_country != 'TW'` → 自動插入「報關」於「包裝出貨」與「物流配送」之間

---

## 3. 預警機制（三層架構）

### 3.1 Layer 1：階段 SLA 計時器

- 每個 `order_stages` 進入時記錄 `entered_at`。
- Worker Cron `alert-scanner` 每 15 分鐘掃 `status='active'` 的階段。
- 計算 `elapsed = now - entered_at`：
  - `elapsed >= sla_yellow_hours` → 黃燈
  - `elapsed >= sla_red_hours` → 紅燈
- 升級時 `INSERT alerts` 並 push 到 notification queue。

### 3.2 Layer 2：物流追蹤異常

- Worker Cron `logistics-poller` 每 60 分鐘輪詢未完成的物流。
- 規則：
  - 24h 無更新 → 黃燈
  - 狀態為「投遞失敗 / 退回」 → 紅燈
- 國內優先用黑貓 API，其他國內貨運記單號（人工或 17track）。
- 國際用 DHL / FedEx 官方 API。

### 3.3 Layer 3：交期承諾預警

- 訂單欄位 `promised_delivery_date`。
- 距交期 ≤ 3 工作日且未到包裝階段 → 黃燈
- 距交期 ≤ 0 工作日且未出貨 → 紅燈

### 3.4 升級鏈（預設值，可配置）

```
黃 alert：
  T=0:00   notify 業務（email + in_app）
  T=0:30   重送 業務（teams）
  T=1:00   升級 sales_manager（teams + email）
  T=2:00   升級 admin（teams + slack + email）

紅 alert：
  T=0:00   notify 業務 + sales_manager（email + teams）
  T=0:30   升級 admin（email + teams + slack）
  T=1:00   未 resolve → 全 admin 群（再推一次）
```

升級時間每租戶可在設定頁微調。

### 3.5 抑制策略

- 同 alert / channel / recipient 24h 內不重送。
- 同 user 5 分鐘內 ≥ 3 條 alert → 切換為摘要模式（每 10 分鐘合併送出）。
- 工作時間（user 設定）外的黃 alert 延後到上班開頭；紅 alert 強制送（可由 user 關閉）。
- 假日靜音：tenant 設國定假日，黃 alert 延後；紅 alert 視 prefs。
- 訂單 completed/cancelled → 取消未送的 alert。

### 3.6 人工觸發 alert

業務 / 主管可在訂單詳情手動點「標紅」並指定通知對象（用於客戶緊急要求等情境）。

---

## 4. 角色與權限

### 4.1 6 種角色

| 角色 | 中文 | 主要職責 |
|------|------|---------|
| `admin` | 管理員 | 全租戶操作、設定、人員 |
| `sales_manager` | 業務主管 | 看本部門訂單、簽核高額、邀請部門業務 |
| `sales` | 業務 | 自己負責的訂單 CRUD、客戶管理 |
| `ops` | 生產 / 倉管 | 階段 4-7、9-11 推進、條碼掃描更新 |
| `accountant` | 會計 | 階段 3、8（打款）、發票上傳、財務報表 |
| `viewer` | 檢視者 | 全部唯讀 |

### 4.2 權限矩陣（節錄關鍵）

| 動作 | admin | sales_mgr | sales | ops | accountant | viewer |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| 看訂單 | 全部 | 本部門全部 | 自己負責 | 出貨中+經手 | 全部 | 全部唯讀 |
| 編輯訂單 | ✅ | 本部門 | 自己 | ❌ | ❌ | ❌ |
| 階段 1, 2, 4 | ✅ | ✅ | ✅ | 4 排程 | ❌ | ❌ |
| 階段 3, 8（打款） | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| 階段 5-7, 9-11 | ✅ | 唯讀 | 唯讀 | ✅ | ❌ | ❌ |
| 高額簽核（>租戶門檻） | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| 上傳發票 / 收款憑證 | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| 財務報表 | ✅ | 本部門 | ❌ | ❌ | 全部 | ❌ |
| 人員管理 | ✅ | 邀本部門業務 | ❌ | ❌ | ❌ | ❌ |
| 看 audit log | ✅ | 本部門 | ❌ | ❌ | 財務相關 | ❌ |
| MCP API token | 發行 | 用自己的 | 用自己的 | 用自己的 | 用自己的 | ❌ |

### 4.3 部門 / 團隊

```sql
teams (
  id, tenant_id FK,
  name,                     -- "業務一部" 等
  manager_user_id FK,
  created_at
)

users.team_id FK            -- 員工屬哪個部門
```

- `sales` 之間：同部門互相可見（依 `users.team_id`）。
- `sales_manager` 看本部門全部。
- 跨部門協作走 `order.shared_with_user_ids` 明確授權。

### 4.4 高額訂單簽核

- 租戶設定 `high_value_threshold`（如 100 萬 NTD）。
- 訂單 `amount > threshold` → 進階段 4 前需 sales_manager 或 admin 簽核。
- 簽核當作 micro-stage 內部記錄（不顯示在主流程進度條上）。
- 通知透過 Teams + Email 推送主管。

### 4.5 代理 / 共享機制

- `admin` 可代任何角色操作，audit_log 標記 `acted_as`。
- 業務出差可暫時 share 訂單給同事（最長 30 天，到期自動撤回）。

---

## 5. 資料模型

### 5.1 主要 Tables（共 15 個）

```
tenants
users
teams
customers
stage_templates
orders
order_items
order_stages
documents
logistics
alerts
notifications
notification_prefs
audit_log
sessions
api_keys              -- Phase 2 MCP
tenant_integrations   -- Teams/Slack webhook、carrier API key 等
```

### 5.2 核心 Schema

```sql
tenants (
  id TEXT PK,                 -- ULID
  name, slug,                 -- slug 用於 URL: /t/<slug>
  plan TEXT,                  -- free/pro/enterprise
  entra_tenant_id,            -- M365 SSO
  high_value_threshold DECIMAL DEFAULT 1000000,
  locale_default DEFAULT 'zh-TW',
  created_at
)

users (
  id, tenant_id FK,
  email,                      -- (tenant_id, email) UNIQUE
  display_name,
  entra_oid,                  -- M365 OID
  role TEXT,                  -- admin/sales_manager/sales/ops/accountant/viewer
  team_id FK NULL,
  status,                     -- active/suspended/invited
  teams_user_id, slack_user_id,
  created_at
)

teams (
  id, tenant_id FK,
  name, manager_user_id FK,
  created_at
)

customers (
  id, tenant_id FK,
  name, tax_id,
  contact_name, contact_phone, contact_email,
  billing_address, shipping_address_default,
  payment_terms,              -- net30/cod/prepaid_30/prepaid_50/...
  credit_limit DECIMAL, currency_default,
  notes, tags JSON,
  is_active, created_at
)

stage_templates (
  id, tenant_id FK,
  name, type,                 -- manufacturing/wholesale
  stages JSON,                -- 階段陣列：name, sla_yellow, sla_red, optional, condition
  is_default
)

orders (
  id, tenant_id FK,
  order_no,                   -- (tenant_id, order_no) UNIQUE
  customer_id FK,
  owner_user_id FK,           -- 負責業務
  template_id FK,
  type TEXT,                  -- manufacturing/wholesale
  status,                     -- active/completed/cancelled/on_hold
  shipping_country,
  amount DECIMAL, currency,
  promised_delivery_date,
  current_stage_idx,
  shared_with_user_ids JSON,
  created_at, completed_at
)

order_items (
  id, tenant_id FK, order_id FK,
  line_no, sku, product_name,
  qty, unit, unit_price, subtotal,
  bom_ref TEXT NULL, notes
)

order_stages (
  id, order_id FK,
  stage_idx, stage_name,
  status,                     -- pending/active/completed/skipped
  entered_at, completed_at,
  sla_yellow_hours, sla_red_hours,
  alert_state TEXT,           -- none/yellow/red
  next_escalation_at,         -- 升級鏈時序
  assignee_user_id, notes
)

documents (
  id, tenant_id FK, order_id FK,
  doc_type,                   -- po/invoice/packing_list/bom/customs/qc/other
  source,                     -- generated/uploaded
  file_key,                   -- R2 key
  file_name, mime_type, size_bytes,
  metadata JSON,
  uploaded_by, created_at
)

logistics (
  id, order_id FK,
  carrier,                    -- tcat/hct/ezship/sf/dhl/fedex/manual
  tracking_no,
  status, last_update_at,
  estimated_delivery_at, delivered_at,
  raw_response JSON, poll_count
)

alerts (
  id, tenant_id FK, order_id FK, stage_id FK,
  severity,                   -- yellow/red
  reason,                     -- sla_exceeded/logistics_stale/promised_date_at_risk/manual
  triggered_at, acknowledged_at, resolved_at,
  acknowledged_by_user_id,
  manual_triggered_by_user_id NULL
)

notifications (
  id, tenant_id FK,
  channel,                    -- email/in_app/teams/slack/mcp
  recipient_user_id,
  alert_id FK NULL,
  template_id, payload JSON,
  status,                     -- queued/sent/failed
  external_id,                -- 第三方訊息 ID
  sent_at, error
)

notification_prefs (
  user_id PK,
  email_enabled, teams_enabled, slack_enabled, inapp_enabled,
  yellow_channels JSON, red_channels JSON,
  digest_enabled,
  work_hours_start, work_hours_end,
  work_days INT,              -- bitmap
  dnd_until, override_red_in_dnd
)

audit_log (
  id, tenant_id FK,
  actor_user_id, acted_as_user_id NULL,
  via TEXT,                   -- web/mobile/mcp/system
  action,                     -- order.create/stage.advance/...
  target_type, target_id,
  before JSON, after JSON,
  ip, user_agent, created_at
)

sessions (
  id, user_id FK,
  device_label,
  expires_at, last_active_at,
  ip, user_agent
)

api_keys (                    -- Phase 2 MCP
  id, tenant_id FK, user_id FK,
  name, scope,                -- read_only/read_write
  key_hash, last_used_at,
  expires_at NULL, created_at, revoked_at NULL
)

tenant_integrations (
  id, tenant_id FK,
  type,                       -- teams/slack/tcat/dhl/fedex/email
  config JSON,                -- 加密 API key、webhook URL
  enabled, last_test_at, created_at
)
```

### 5.3 多租戶資料隔離

- 所有非系統表都帶 `tenant_id`。
- Workers middleware 解析 SSO session 後，將 `tenantId` 注入請求 context。
- DB 層用 Drizzle 的 query helper 強制 `WHERE tenant_id = ?`，漏帶會 throw。
- 自動測試覆蓋：每個 endpoint 跑「非本租戶 user 嘗試取資料 → 必須回 404/403」。

### 5.4 多租戶資料庫策略（漸進式）

```
Stage 1（≤ 50 租戶）：       單一 D1 + tenant_id 欄位
Stage 2（50-500 租戶）：     sharded D1，按 tenant_id hash 分到 N 個 D1
Stage 3（特定大客戶要求隔離）：每租戶獨立 D1，KV 路由
```

`getDb(tenantId)` helper 抽象化，從 1 升 2/3 不改 query 邏輯。

---

## 6. 通知派送機制

### 6.1 整體流程

```
[CRON] alert-scanner（15min） ──┐
                                ├──▶ [Queue: notifications]
[CRON] logistics-poller（60min）─┤
                                │
[App] 手動觸發 / 階段事件     ──┘
                                          │
                                          ▼
                              [Worker: notification-dispatcher]
                                          │
                ┌─────────┬───────────────┼───────────────┬─────────┐
                ▼         ▼               ▼               ▼         ▼
              email     in_app          teams           slack      mcp
           (Resend)    (DO + WS)    (Graph API)    (chat.postMsg)  (Phase 2)
```

### 6.2 Channel Adapter 介面

```ts
interface NotificationAdapter {
  send(input: {
    user: User;
    template: TemplateId;
    vars: Record<string, unknown>;
    locale: string;
  }): Promise<{ ok: boolean; externalId?: string; error?: string }>;
}
```

新增 LINE / Telegram / 自家 webhook 只需實作此介面並註冊。

### 6.3 Email

- Resend API（或 Mailchannels via Workers）。
- 模板（i18n）：`alert-yellow`, `alert-red`, `stage-advanced`, `order-completed`, `digest`。
- 失敗自動 fallback 到備用 SMTP（如 SendGrid）。

### 6.4 Teams

- **第一階段使用 Bot Framework + Graph API**，**僅頻道通知**（不私訊）。
- 訂單預警送到租戶設定的指定頻道（`tenant_integrations.config.teams_channel_id`）。
- Adaptive Card 格式，含「查看訂單」「認領處理」按鈕。
- 認領按鈕透過 invoke action 回呼系統 API。

### 6.5 Slack

- Slack Bolt SDK 或直接 `chat.postMessage`。
- Block Kit 格式，與 Teams Adaptive Card 結構等價。
- 同樣只發頻道（不私訊）。

### 6.6 In-App

- D1 寫一筆 → Durable Object push WebSocket → 前端 toast + bell badge。

### 6.7 失敗重試

- Cloudflare Queue 指數退避：1m → 5m → 15m → 1h → 6h，5 次後丟 DLQ。
- 第三方 503 → 自動降級補送 email。

### 6.8 Idempotency

通知 payload dedup key = `alertId + channel + escalation_step`，KV TTL 24h。

---

## 7. 文件管理

### 7.1 6 類文件支援

| 類型 | doc_type | 來源 | 結構化欄位 |
|------|----------|------|----------|
| 訂單 / 報價單 | `po` | generated（系統表單填寫） | order_no, customer, items, amount |
| 發票 / 出貨單 | `invoice` | generated | invoice_no, items, tax, total |
| 裝箱單 | `packing_list` | generated | boxes, items per box, weight |
| BOM / 工單 | `bom` | uploaded | metadata.parts JSON |
| 報關文件（B/L、Invoice、PL） | `customs` | uploaded | metadata.bl_no, hs_code |
| QC 報告 | `qc` | uploaded | metadata.test_results |

### 7.2 系統產生 PDF

- PO / Invoice / Packing List / 出貨單：用 `@react-pdf/renderer` 或 `pdfmake`。
- 多語模板（zh-TW / en）。
- 出貨單自動印 QR（角落 2cm × 2cm，payload = `https://app/o/<id>?t=<hmac>`）。

### 7.3 上傳

- 直傳 R2（Workers presigned URL，避免大檔過 Worker CPU 限制）。
- 檔案類型限制：pdf / jpg / png / xlsx / csv / docx。
- 單檔 ≤ 25 MB。
- 病毒掃描（可選 ClamAV 服務或先省略）。

---

## 8. 物流串接

### 8.1 國內

- **黑貓宅急便**：官方 API（須申請），輪詢 GET 取狀態。
- **新竹 / 宅配通 / 順豐（台灣段）**：無公開 API → 人工貼單號 + 17track 統一查詢。
- **手動模式**：所有不支援 API 的單號可用此模式，倉管手動標記「已簽收」。

### 8.2 國際

- **DHL Express API**：tracking + label。
- **FedEx Web Services**：tracking。
- **17track.net**：統一查詢備援（多 carrier）。

### 8.3 carrier 抽象

```ts
interface CarrierAdapter {
  carrier: string;
  pollStatus(trackingNo: string): Promise<LogisticsStatus>;
  estimateDelivery?(...): Promise<Date>;
}
```

每個 carrier 實作此介面，dispatcher 依 `logistics.carrier` 路由。

---

## 9. SSO 與身份

### 9.1 Microsoft Entra ID（OIDC）

```
[Browser] /auth/login
   ↓ 302 → Microsoft login
[M365] 使用者輸入帳密
   ↓ /auth/callback?code=...
[Workers]
   1. PKCE 換 access_token + id_token
   2. 驗 id_token JWT (jose, JWKS)
   3. 取 oid / email / tid / name
   4. UPSERT user by (tenant_id, entra_oid)
   5. 第一次登入：role=viewer，通知 admin
   6. 簽 session JWT (HS256, 7d, HttpOnly cookie)
```

### 9.2 多租戶配置

- 每租戶在設定填自己的 `entra_tenant_id`。
- 多租戶 SaaS 模式用 multi-tenant Azure App，OAuth authority `common`，依 `tid` 對應或自助建立租戶。

### 9.3 預留帳密登入路徑

架構保留 password / magic link，第二階段才開放（給沒有 M365 的客戶）。

---

## 10. PWA 與行動端

### 10.1 PWA 設定

- `manifest.webmanifest`（standalone, portrait, theme color, shortcuts: 掃條碼 / 我的待辦）
- Service Worker：app shell precache + API GET stale-while-revalidate
- Web Push：iOS 16.4+ 須加到主畫面才能推；Android 直接支援
- 無離線寫入（mutating 操作必須在線）

### 10.2 條碼 / QR

**訂單 QR**（系統產生，印在出貨單上）：
- payload: `https://app/o/<orderId>?t=<short_token>`
- `short_token = HMAC(orderId + tenant_secret).slice(0, 8)`

**SKU 條碼掃描**（揀貨用）：
- 訂單明細支援逐項掃描勾選
- 連續掃描模式

### 10.3 掃描技術

- 首選：`BarcodeDetector` API（Chrome/Edge Android、Safari iOS 17+）
- iOS 16 不支援 → **不做 fallback**（user share < 5%，要求升級或用桌面）
- Torch API、震動回饋、音效

### 10.4 倉管工作流

```
1. 開手機（已 SSO，session 14d）
2. 看「待包裝 3 筆」
3. 掃訂單條碼 → 進詳情
4. 揀貨：對 SKU 條碼勾選
5. 拍出貨照（1-3 張） → 自動上傳 R2
6. 點「完成本階段」 → 自動進下一階段
7. 輸入 / 掃物流單號 → 系統開始輪詢
8. 訂單從待辦消失
```

---

## 11. Microsoft Teams MCP 整合

### 11.1 Phase 1：通知（必做，已收斂在 §6.4）

僅頻道通知，Adaptive Card with action buttons。

### 11.2 Phase 2：MCP Server

#### 部署

獨立 Worker：`mcp.<domain>`，使用 `@modelcontextprotocol/sdk`。

#### 認證：OAuth Device Flow

```
1. AI client (Claude Desktop / Teams Copilot) 啟動 MCP 連線
2. Server 回 device_code + user_code + verification_uri
3. User 在瀏覽器登入系統並輸入 user_code 授權
4. AI client 收到 access_token (短 token + refresh)
5. 後續請求帶 Bearer token
```

#### 8 個工具

| 工具 | 描述 | 權限 |
|------|------|------|
| `list_orders(filter, limit)` | 查詢訂單清單 | read |
| `get_order(orderId)` | 取單張訂單詳情 | read |
| `get_order_alerts(scope)` | 取 alert 清單 | read |
| `search_customer(query)` | 找客戶 | read |
| `advance_stage(orderId, note, confirm)` | 推進階段 | write |
| `add_note(orderId, content)` | 加評論 | write |
| `get_logistics(orderId)` | 取物流狀態 | read |
| `upload_document(orderId, file_b64, doc_type)` | 上傳文件 | write |

#### 資源（Resources）

- `order:///<id>` — 完整訂單
- `alerts:///today` — 今日 alert
- `reports:///sla` — SLA 報表

#### 安全

- 共用系統 RBAC（`canAccessOrder` 等同 web 版）。
- Token scope ≤ user 本身權限。
- 每次工具呼叫寫 audit_log，標 `via='mcp'`。
- Rate limit：60 calls/min/token，超出 429。
- write 工具強制 `confirm: true` 參數，避免 LLM 誤觸發。

---

## 12. 技術棧

### 12.1 前端

- React 18 + TypeScript + Vite（PWA plugin）
- Tailwind CSS + shadcn/ui
- TanStack Query（server state）+ Zustand（client state）
- TanStack Router
- react-hook-form + zod
- dnd-kit（看板拖拉）
- i18next（zh-TW / en）

### 12.2 後端

- Cloudflare Workers + Hono
- D1 + Drizzle ORM
- R2（檔案）/ KV（session、idempotency）/ Queues / Durable Objects
- jose（JWT）
- @microsoft/microsoft-graph-client / @slack/web-api
- @modelcontextprotocol/sdk（Phase 2）

### 12.3 工具

- bun（runtime + pkg manager）
- wrangler（部署）
- drizzle-kit（migrations）
- Vitest + Playwright

---

## 13. 目錄結構

```
orderflow/
├── apps/
│   ├── web/                    # Pages 前端
│   │   └── src/
│   │       ├── routes/
│   │       ├── features/
│   │       │   ├── orders/ customers/ stages/ alerts/
│   │       │   ├── documents/ reports/ scanner/ settings/
│   │       ├── components/ui/
│   │       ├── lib/
│   │       └── i18n/
│   └── api/                    # Workers
│       └── src/
│           ├── routes/ services/ db/
│           ├── auth/ notifications/ logistics/ crons/
│           └── mcp/            # Phase 2
├── packages/
│   ├── shared/                 # types & zod schemas
│   ├── ui-tokens/
│   └── notifications-templates/
└── docs/
    └── superpowers/specs/
```

---

## 14. 部署架構

```
[Cloudflare DNS]
      │
      ├─▶ pm.app (Pages)        前端 SPA + PWA
      ├─▶ api.pm.app (Workers)  REST API + Cron + Queue + DO
      └─▶ mcp.pm.app (Workers)  Phase 2 MCP Server

外部服務：
  - login.microsoftonline.com (SSO)
  - graph.microsoft.com (Teams)
  - slack.com/api
  - 黑貓 / DHL / FedEx APIs
  - resend.com (Email)
```

### 14.1 開發階段（Phase 0-1）

僅在 localhost 跑：
- `apps/web` 用 `bun run dev`（Vite, port 5173）
- `apps/api` 用 `wrangler dev` 或 `bun run dev`（port 8787）
- D1 用 wrangler local SQLite
- R2 用 wrangler local emulator

### 14.2 公網部署（Phase 2 後決定）

多租戶 URL 預設用路徑：`pm.app/t/<slug>`（不需 wildcard cert）。
Domain 名稱 / 註冊在 Phase 2 末再決定。

---

## 15. 開發路線圖

### Phase 0：地基（2 週）

- 專案 monorepo + workspace + CI
- D1 schema + initial migrations
- M365 SSO 端到端
- UI shell（layout、navigation、i18n）
- 多租戶 middleware + RBAC framework

### Phase 1：MVP（5-6 週）

- W1：客戶管理 + 訂單 CRUD + 訂單明細
- W2：階段引擎（templates + advance + history）
- W3：文件上傳 + 系統產生 PO/Invoice/PL（PDF）
- W4：預警 cron + 通知抽象 + Email adapter
- W5：Teams + Slack adapter + 升級鏈
- W6：物流串接 + 看板 UI + 報表

> **MVP 結束內部可上線**

### Phase 2：行動 + 進階（3-4 週）

- W1：PWA 配置 + 行動端 RWD
- W2：條碼掃描 + SKU 揀貨
- W3：高級權限（業務主管、會計專屬視圖、簽核流）
- W4：報表 / AR aging / SLA 統計

### Phase 3：MCP + SaaS 化（3-4 週）

- W1：MCP server 設計 + OAuth Device Flow
- W2：8 個工具實作 + audit + rate limit
- W3：Teams Copilot manifest + Azure App
- W4：多租戶自助註冊流（外賣選用）

**總計：13-16 週**（單人全職）

---

## 16. 風險與緩解

| 風險 | 緩解 |
|------|------|
| 黑貓 API 申請流程慢 | 先支援手動單號，API 補上 |
| Teams Bot Azure 註冊耗時 | Phase 1 用 Incoming Webhook 過渡 |
| D1 大量 cron 撞 CPU 限制 | 分批 + ulid 分頁；單次 ≤ 100 訂單 |
| iOS PWA 推播限制 | 引導使用者「加到主畫面」 |
| 多租戶資料外洩 | 中央 middleware + DB row-level + 自動測試 |
| MCP 工具誤操作 | write 工具強制 `confirm` + dry-run + audit |

---

## 17. 規模上限對照

| 指標 | 此架構撐到 |
|------|----------|
| 租戶數 | 50（單 D1）／ 500（sharded） |
| 單租戶員工數 | ~200 |
| 單租戶訂單 / 月 | ~5,000 |
| 並行 alert | 數千 / 分鐘 |
| 月成本（內部用、單租戶） | $5-15 USD |
| 月成本（50 租戶 SaaS） | $80-200 USD |

---

## 18. 已決定的設計選擇（決議記錄）

| 議題 | 決議 |
|------|------|
| 系統定位 | 多租戶 SaaS 架構，先內部用 |
| 產業 | 製造 + 批發混合 |
| 階段彈性 | 系統範本 + 租戶可微調 |
| 規模目標 | 數十並行訂單（CF Pages + D1） |
| 物流 | 國內為主（黑貓 API），國際少量（DHL / FedEx） |
| 通知管道 | Email + Teams + Slack |
| 角色 | 6 種：admin / sales_manager / sales / ops / accountant / viewer |
| SSO | Microsoft Entra ID（M365） |
| 文件類型 | 6 類全要，系統產生 + 上傳混合 |
| 行動端 | PWA + 條碼掃描，無離線 |
| Teams MCP | Phase 1 通知（僅頻道）／ Phase 2 MCP Server |
| MCP 認證 | OAuth Device Flow |
| 階段升級時間 | 系統預設 + 可手動調整 |
| 批次摘要通知 | 啟用 |
| 紅 alert in DND | 強制送 |
| 人工觸發 alert | 支援 |
| 業務主管角色 | 加入（5 → 6） |
| 會計角色 | 加入 |
| 客戶外部介面 | 不做 |
| 訂單 QR 位置 | 出貨單 PDF 角落 2cm × 2cm |
| SKU 揀貨 | 支援 |
| iOS 16 fallback | 不做 |
| MCP 工具集 | 8 個（read 5 + write 3） |
| 路線圖 | 13-16 週，照 Phase 0/1/2/3 順序 |
| 第一階段部署 | localhost only |
| 公網 URL 結構 | 路徑型 `pm.app/t/<slug>`（Phase 2 末再啟用） |

---

## 19. 後續步驟

1. User review 此 spec
2. 進入 implementation plan（writing-plans skill）
3. Phase 0 啟動
