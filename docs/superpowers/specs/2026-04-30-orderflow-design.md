# OrderFlow — 醫療器材場內部訂單與品質管理系統設計規格

- 文件版本：**v0.2（QMS / Part 11 重新定位）**
- 撰寫日期：2026-04-30
- 撰寫者：swlucifer@gmail.com（與 Claude 協作）
- 狀態：待 user review，後續進入 implementation plan
- 變更摘要（v0.1 → v0.2）：移除 multi-tenant SaaS 定位；加入 ISO 13485 + TFDA + FDA 21 CFR 820 + 21 CFR Part 11 合規範圍；加入 Lot/UDI/Serial 追溯、電子簽章、hash-chained 稽核軌跡、NCR/CAPA/客訴/文件管制模組；新增軟體驗證與資料保留章節

---

## 1. 系統定位

從訂單接收到出貨完成的**內部**訂單與品質管理系統，整合 ISO 13485 / 21 CFR 820 要求的生產／驗收／追溯／不符合品／CAPA／客訴流程。

- **使用對象**：本公司內部業務、生產、倉管、品保（QA/QC）、會計、主管、稽核員。**不**對外開放。
- **產業屬性**：自有設計製造（own design and manufacture）的 Class II 醫療器材製造商。
- **法規目標**：
  - ISO 13485:2016（國際 QMS 主軸）
  - 衛福部食藥署（TFDA）醫療器材品質管理系統準則（GMP）
  - FDA 21 CFR Part 820（Quality System Regulation, QSR — 美國輸入）
  - FDA 21 CFR Part 11（Electronic Records / Electronic Signatures）
- **不在範圍內**：
  - 對外公開租戶註冊／SaaS／訂閱付費
  - 設計管制（DHF）— 由獨立 PLM/QMS 系統處理，本系統可建立索引連結但不儲存設計記錄
  - 校驗管理（calibration）— 連結現有外部系統，不重做
  - 服務維修記錄（820.200）— v1 不實作，預留資料模型
  - 統計技術（820.250）— v1 不實作

### 1.1 核心需求

1. 同時追蹤多筆訂單從接單到出貨的完整生命週期。
2. 每階段預警機制，含 SLA／物流異常／交期承諾／NCR 結案／CAPA 結案／客訴法定通報期限。
3. 多角色權限，含**職責分離（separation of duties）**：建單者不得自行核可同筆訂單超過閾值。
4. Web 為主、行動端 PWA 支援，含 UDI/Lot 條碼掃描（驗收、揀貨、最終檢驗）。
5. **Microsoft Teams 整合（內部用）**：staff 透過 Teams Copilot 查詢訂單狀態與簡單寫入操作；MCP server 強制 OAuth Device Flow 並對 critical write 強制電子簽章。
6. **QMS 整合**：
   - 不符合品（NCR, 820.90）登錄與處置簽核
   - CAPA（820.100）流程與效益評估
   - 客訴（820.198）登錄、調查、結案、TFDA/FDA 通報旗標
   - 文件管制（820.40 / Part 11 11.10(k)）SOP/WI 版本與審核
   - 訓練記錄（820.25）對應角色授權
7. **電子簽章**（Part 11 11.50 / 11.70 / 11.100 / 11.200 / 11.300）覆蓋所有 GxP-critical 寫入操作。
8. **稽核軌跡不可竄改**（Part 11 11.10(e)）：hash-chain 連結每筆 audit log。
9. **年度 QMS 稽核報表**：可一鍵彙整管理審查需要的 KPI、NCR/CAPA 趨勢、權限變更、訓練覆蓋。

### 1.2 規模假設

- 並行訂單：~數十筆
- 員工：≤ 200
- 月訂單量：≤ 1,000（醫療器材製造節奏）
- 月 NCR：≤ 50
- 月客訴：≤ 30
- 法定保留期：**7 年**（醫療器材記錄）

### 1.3 不採用的方案（與 v0.1 差異）

| v0.1 規劃 | v0.2 變更 | 原因 |
|----------|---------|------|
| Multi-tenant 資料庫，所有 table 帶 `tenant_id` | 單一公司部署，移除 `tenant_id` 欄位 | QMS 系統屬於受管制設備，不混租 |
| 自助註冊 + 計畫收費 | 不開放 | 內部系統 |
| 公開 URL（`api.orderflow.example.com`）| 內網 + Cloudflare Access Zero Trust 閘道 | 醫療器材資安要求 |
| 一般 audit_log（append-only） | hash-chain immutable audit_log + RFC 3161 timestamp（可選） | Part 11 11.10(e) |
| Stage 推進為單純權限 check | Critical stage 需 e-sig + 雙因素 | Part 11 11.200 |
| 訂單品項只有 SKU/qty | 訂單品項加上 Lot/UDI/Serial/製造日/有效日 | 820.60 / 820.65 / UDI Rule |

---

## 2. 法規範圍對照表

每個系統功能需明確對應到法規條文，作為 IQ/OQ/PQ 驗證 traceability matrix 的基礎。

| 法規條款 | 系統實作位置 | 驗證方式 |
|---------|-------------|---------|
| **ISO 13485 §4.1.6** 軟體驗證 | Validation Master Plan + IQ/OQ/PQ | 文件 + 測試報告 |
| **ISO 13485 §4.2.4 / §4.2.5** 文件控制與記錄 | Document Control 模組 + audit_log retention | 系統測試 |
| **ISO 13485 §6.2** 人員能力／訓練 | Training Records 模組 | 系統測試 |
| **ISO 13485 §7.5.8** 識別與追溯 | Lot/UDI/Serial fields + DHR | 系統測試 |
| **ISO 13485 §8.3** 不合格品 | NCR 模組 | 系統測試 |
| **ISO 13485 §8.5** 改善 | CAPA 模組 | 系統測試 |
| **21 CFR 820.25** 人員 | Training Records | 系統測試 |
| **21 CFR 820.40** 文件管制 | Document Control 模組 | 系統測試 |
| **21 CFR 820.60** 識別 | Lot/UDI/Serial | 系統測試 |
| **21 CFR 820.65** 追溯 | Lot trace forward + backward query | 系統測試 |
| **21 CFR 820.70(i)** 製程控制軟體驗證 | VMP + IQ/OQ/PQ | 文件 |
| **21 CFR 820.80** 接收／製程中／最終驗收 | Stage acceptance records + e-sig | 系統測試 |
| **21 CFR 820.86** 驗收狀態 | order_items.acceptance_status | 系統測試 |
| **21 CFR 820.90** 不合格品 | NCR 模組 | 系統測試 |
| **21 CFR 820.100** CAPA | CAPA 模組 | 系統測試 |
| **21 CFR 820.180** 一般記錄要求 | Retention policy + export | 系統測試 |
| **21 CFR 820.184** DHR | DHR 自動彙整 | 系統測試 |
| **21 CFR 820.186** DMR 索引 | 連結至外部 PLM | 設計確認 |
| **21 CFR 820.198** 客訴檔 | Complaint 模組 + MDR/Vigilance flag | 系統測試 |
| **21 CFR Part 11 §11.10(a)** 系統驗證 | VMP | 文件 |
| **21 CFR Part 11 §11.10(b)** 可生成副本 | Order/NCR/CAPA 全 PDF + CSV 匯出 | 系統測試 |
| **21 CFR Part 11 §11.10(c)** 記錄保護 | 7 年保留 + R2 immutable bucket + 異地備份 | 系統測試 + IT 政策 |
| **21 CFR Part 11 §11.10(d)** 存取限制 | M365 SSO + MFA + Cloudflare Access + RBAC | 系統測試 |
| **21 CFR Part 11 §11.10(e)** 安全稽核軌跡 | Hash-chained audit_log（read-only API） | 系統測試 |
| **21 CFR Part 11 §11.10(f)** 操作順序檢查 | Stage state machine（不可跳階） | 系統測試 |
| **21 CFR Part 11 §11.10(g)** 權限檢查 | RBAC matrix | 系統測試 |
| **21 CFR Part 11 §11.10(h)** 裝置檢查 | Cloudflare Access device posture（Phase 5） | 系統測試 |
| **21 CFR Part 11 §11.10(i)** 訓練 | Training Records 連結 | 系統測試 |
| **21 CFR Part 11 §11.10(j)** 責任政策 | 連結至 SOP（人員手冊） | 文件 |
| **21 CFR Part 11 §11.10(k)** 系統文件管制 | Document Control 模組 | 系統測試 |
| **21 CFR Part 11 §11.50** 簽章呈現 | E-sig 顯示姓名／日期／意涵於記錄旁 | 系統測試 |
| **21 CFR Part 11 §11.70** 簽章與記錄連結 | esig FK + 不可移除 | 系統測試 |
| **21 CFR Part 11 §11.100** 一般要求 | M365 帳號唯一不重用 | M365 + 系統測試 |
| **21 CFR Part 11 §11.200** 簽章組件 | M365 SSO 身分（component 1）+ 密碼再驗證（component 2） | 系統測試 |
| **21 CFR Part 11 §11.300** ID/密碼管制 | M365 password policy + MFA + 帳號鎖定 | M365 設定 |

---

## 3. 角色、權限與職責分離

### 3.1 角色清單（6 角色）

| 角色 code | 中文名 | 範圍 | 主要職責 |
|----------|-------|------|---------|
| `admin` | 系統管理員 | 全系統 | 帳號／角色／模板／系統設定（受 access review 管制） |
| `qa_manager` | 品保主管 | 全系統 | NCR/CAPA/客訴 結案核可、release for shipment 簽章、年度稽核報表 |
| `sales_manager` | 業務主管 | 自管團隊 | 高金額訂單核可、團隊 KPI |
| `sales` | 業務 | 自身訂單 | 接單／報價／客戶溝通 |
| `ops` | 生產／倉管／QC 操作員 | 全訂單 | 生產推進、揀貨、條碼掃描、In-process check |
| `accountant` | 會計 | 全訂單金流 | 收款登錄、發票、AR aging |
| `viewer` | 審計／訪客 | 全系統 read-only | 唯讀（內外部稽核員、外部 ISO/FDA 稽核員臨時帳號） |

> v0.1 的 6 角色擴為 7 角色：新增 `qa_manager`，因為 QMS critical 簽章不能下放到 `admin`（admin 主要是 IT 角色，與品質決策應分離）。

### 3.2 權限矩陣（節錄關鍵動作）

| 動作 | admin | qa_manager | sales_manager | sales | ops | accountant | viewer |
|------|-------|-----------|--------------|-------|-----|-----------|--------|
| 建立／修改訂單 | ✓ | – | ✓ | own | – | – | – |
| 核可訂單（≥ NT$500k） | – | – | ✓ | – | – | – | – |
| 推進一般階段 | ✓ | ✓ | ✓ | own | ✓ | – | – |
| **QC 通過（e-sig）** | – | ✓ | – | – | ✓\* | – | – |
| **Release for shipment（e-sig）** | – | ✓ | – | – | – | – | – |
| 建立 NCR | ✓ | ✓ | – | – | ✓ | – | – |
| **NCR disposition（e-sig）** | – | ✓ | – | – | – | – | – |
| 建立 CAPA | ✓ | ✓ | – | – | – | – | – |
| **CAPA effectiveness 結案（e-sig）** | – | ✓ | – | – | – | – | – |
| 登錄客訴 | ✓ | ✓ | ✓ | ✓ | – | – | – |
| **客訴結案（e-sig）+ MDR 通報判定** | – | ✓ | – | – | – | – | – |
| 上傳／核可 quality document（e-sig） | ✓ | ✓ | – | – | – | – | – |
| 登錄付款 | – | – | – | – | – | ✓ | – |
| 帳號／角色管理 | ✓ | – | – | – | – | – | – |
| 季度 access review 簽核 | – | ✓ | – | – | – | – | – |
| 年度稽核報表產出 | ✓ | ✓ | – | – | – | – | view only |
| 唯讀 audit log | ✓ | ✓ | – | – | – | – | ✓ |

\*ops 僅限執行 QC 檢查（記錄結果），但 release decision（通過 / 不通過 → 是否進入 NCR）需 `qa_manager` e-sig。

### 3.3 職責分離（Separation of Duties, SoD）規則

- **規則 SoD-1**：建單人（`orders.created_by`）不得作為同筆訂單的審批人（`orders.approved_by`）。
- **規則 SoD-2**：NCR 建立人（`ncr.reported_by`）不得作為同筆 NCR 的處置決策人（`ncr.disposed_by`）。
- **規則 SoD-3**：CAPA owner（`capa.owner_id`）不得作為同筆 CAPA 的 effectiveness 結案人（`capa.closed_by`）。
- **規則 SoD-4**：Document 撰寫者（`documents.created_by`）不得作為同份文件審核者（`documents.approved_by`）。
- 系統於 transaction 層強制執行；違反時回 `409 Conflict` + audit log。

### 3.4 季度 access review

- 每季度第 1 個工作日，`access-review-cron` 自動：
  1. 產出每位 active user 的當前角色＋上季登入次數＋上季敏感操作數
  2. 寄信給 `qa_manager` + `admin` 要求逐人 confirm
  3. 90 天未 confirm 的帳號自動 `disabled`（重新啟用需 e-sig）
- 結果儲存於 `access_reviews` 表，作為年度稽核證據。

### 3.5 帳號生命週期

- 入職：HR 提交→`admin` 建立 M365 帳號→指派系統角色（記錄 training 完成才解鎖）
- 異動：角色變更需 `admin` + `qa_manager` 雙簽（e-sig）
- 離職：HR 通知 24 小時內 `admin` disable（systemd cron 每 4h scan M365 disabled list）

---

## 4. 訂單資料模型與追溯（DHR 基礎）

### 4.1 訂單品項擴充欄位

每筆 `order_items` row 對應一個製造批次（lot）或一個序號單元，必須具備：

| 欄位 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `id` | TEXT (ULID) | ✓ | – |
| `order_id` | FK | ✓ | – |
| `sku` | TEXT | ✓ | DMR 產品代碼 |
| `name` | TEXT | ✓ | – |
| `qty` | INT | ✓ | – |
| `unit_price` | DECIMAL | ✓ | – |
| `lot_no` | TEXT | ✓\* | 製造批號（820.60） |
| `udi_di` | TEXT | ✓\* | UDI Device Identifier |
| `udi_pi` | TEXT | – | UDI Production Identifier（lot/serial/exp 編入） |
| `serial_no_range` | TEXT | – | 若為單台序號管制：起訖序號（"SN001-SN010"） |
| `manufacture_date` | DATE | ✓\* | – |
| `expiry_date` | DATE | – | 若有 shelf life |
| `acceptance_status` | ENUM | ✓ | `pending` / `in_process` / `accepted` / `rejected` / `quarantine`（820.86） |
| `qc_inspector_id` | FK users | – | – |
| `qc_signed_at` | TIMESTAMP | – | – |
| `qc_esig_id` | FK e_signatures | – | Part 11 簽章連結 |

\*在訂單品項離開「採購備料」階段時必填。建立階段可暫存空值。

### 4.2 DHR（Device History Record）自動彙整

每筆訂單可一鍵產生符合 820.184 的 DHR PDF，內含：
1. 製造日期（`manufacture_date` 範圍）
2. 製造數量（sum `qty`）
3. 出貨／放行數量
4. 驗收記錄（每個 stage acceptance + e-sig）
5. Primary identification label（從 documents 中 `doc_type=label` 抽取）
6. Device identification（UDI DI + PI）+ Lot / Serial 控制號
7. NCR 連結（若有）
8. 出貨記錄與簽章

### 4.3 追溯查詢（820.65）

- **Forward trace**：給 lot_no，列出所有出貨給哪些客戶／訂單／日期。
- **Backward trace**：給訂單 → 列出所用 lots → （v2 連動 ERP 列出原料來源）。
- 提供 CSV／PDF 匯出（recall scenario 演練）。

---

## 5. 階段範本（QMS-aware）

延續 v0.1 的 11/7 stage 範本，但每個 stage 補上：
- `requires_acceptance_record`：bool — 是否寫入 `acceptance_records`
- `requires_esig`：bool — 推進前需 Part 11 e-sig
- `esig_meaning`：enum `review` / `approval` / `responsibility` / `authorship`
- `qms_clause`：對應法規條款字串（用於追溯 traceability matrix）

### 5.1 製造業範本（11 階段，QMS 增強）

| # | stage_key | 中文名 | acceptance | e-sig | meaning | QMS 條款 |
|---|-----------|-------|-----------|------|---------|---------|
| 1 | `received` | 訂單確認 | – | – | – | – |
| 2 | `approval` | 客戶簽核 | – | – | – | – |
| 3 | `deposit_paid` | 定金 | – | – | – | – |
| 4 | `incoming_inspection` | 來料檢驗 | ✓ | ✓ | review | 820.80(a) |
| 5 | `production_planning` | 生產排程 | – | – | – | – |
| 6 | `production` | 生產製造 | ✓ | – | – | 820.70 |
| 7 | `in_process_qc` | 製程中檢驗 | ✓ | ✓ | review | 820.80(b) |
| 8 | `final_qc` | 最終檢驗 | ✓ | ✓ | approval | 820.80(c) |
| 9 | `release_for_shipment` | 出貨放行 | ✓ | ✓ | **approval** | 820.80(d), 820.184 |
| 10 | `final_paid` | 尾款 | – | – | – | – |
| 11 | `packed_shipped` | 包裝出貨 | – | ✓ | responsibility | – |
| – | `customs` | 報關 | – | – | – | 條件式 |
| – | `delivered` | 物流送達 | – | – | – | 系統自動 |

> v0.1 11 stages 重組：拆出 `incoming_inspection`、`in_process_qc`、`final_qc`、`release_for_shipment` 四個獨立 acceptance stage，符合 820.80。

### 5.2 批發 / 經銷範本

純通路，**移除**（公司是自有設計製造，不做純批發）。若未來新增 OEM 代工子公司，再加回。

### 5.3 階段範本管制

- 範本本身屬於 quality document：修改需 `admin` + `qa_manager` e-sig，建立新版本，舊版鎖定，已開單者沿用舊版（記錄 `orders.stage_template_version`）。
- 不允許「直接編輯目前範本」— 一律 clone → 修改 → 簽核 → 發布。

---

## 6. 預警機制（五層）

延續 v0.1 三層 + 新增兩層 QMS 警示。

### 6.1 Layer 1：階段 SLA 計時器

同 v0.1。SLA 改寫入 `stage_template.stages_json` 而非 hardcode。

### 6.2 Layer 2：物流追蹤異常

同 v0.1。

### 6.3 Layer 3：交期承諾預警

同 v0.1。

### 6.4 Layer 4：NCR 結案逾期（NEW）

- NCR `target_disposition_date` 距今 ≤ 3 工作日 → 黃
- 已逾期未 disposed → 紅，升級至 `qa_manager`
- 預設 disposition 期：14 工作日（可設定）

### 6.5 Layer 5：CAPA / 客訴法定通報期限（NEW）

- CAPA effectiveness review 期限預設 90 天；逾期紅燈
- **客訴 + 嚴重不良反應旗標**：依 TFDA 醫療器材通報辦法／FDA 21 CFR 803 MDR：
  - **死亡 / 嚴重傷害**：30 日內 → 距 deadline 7 日黃、距 deadline 1 日紅、逾期立即 page admin + qa_manager
  - **故障可能造成死亡 / 嚴重傷害**：30 日內，同上
  - **其他**：依個案

### 6.6 升級鏈

延續 v0.1。對於 QMS critical alert，預設升級對象變為 `qa_manager` 而非 `sales_manager`。

---

## 7. 通知頻道

延續 v0.1：Email（Resend）+ Microsoft Teams（Graph API channel webhook）+ Slack（incoming webhook）+ Web Push（PWA）。

QMS 變更：
- **客訴 MDR 30 日警示**強制走 Email + Teams + Slack 三路同時送（避免單通道失效）
- **Release for shipment 簽章**完成後自動通知 sales（你的訂單可以出貨了）

---

## 8. 認證、SSO、存取控制

### 8.1 雙層閘道

```
Internet
   ↓
[Cloudflare Access — Zero Trust gate]
   identity = M365 Entra ID + MFA enforced + Device posture (Phase 5)
   ↓
[Cloudflare Pages / Workers]
   application JWT (HS256, HttpOnly cookie, 2h absolute, 30min idle)
   ↓
[D1 + R2 + KV + Queues]
```

- Cloudflare Access 是第一道：未通過直接 403，沒到應用 layer
- 應用 layer 仍走 OIDC PKCE 取得 ID token → 換取 application session
- 雙層的好處：即使應用 JWT 被偷，沒有 Cloudflare Access cookie 也進不來；反之亦然

### 8.2 Session 政策

- Idle timeout：30 分鐘無動作自動登出
- Absolute timeout：2 小時必須重新 SSO（critical 操作之間 ≤ 2h，符合 Part 11 11.200(b) 同 session 簽章）
- 並行裝置限制：3（沿用 artisebio-web 現有政策）
- 強制重認證觸發：
  - 任何 e-sig 操作（無論 idle 多久都要密碼）
  - admin 角色管理頁進入
  - access review 簽核

### 8.3 MFA

- 由 M365 Entra ID 強制（Conditional Access policy）
- 應用層不重做，但會讀 ID token `amr` claim 確認確實有 `mfa`，否則拒絕

### 8.4 IP / 設備管制（Phase 5）

- Cloudflare Access policy：只允許公司 VPN 出口 IP + 已註冊裝置（EDR 確認）
- 訪客 viewer 帳號（外部稽核員）：時效式 magic link，不走 Entra；24h 過期，全程 read-only

---

## 9. 電子簽章（21 CFR Part 11 完整實作）

### 9.1 何時需要 e-sig

| 動作 | meaning |
|------|---------|
| Incoming inspection 通過／不通過 | review |
| In-process QC 通過／不通過 | review |
| Final QC 通過／不通過 | approval |
| **Release for shipment** | approval |
| 出貨包裝確認 | responsibility |
| NCR disposition（rework / scrap / use-as-is / return） | approval |
| CAPA closure（effectiveness verified） | approval |
| 客訴結案 + MDR 判定 | approval |
| Quality document 審核發布 | approval |
| 階段範本變更發布 | approval |
| 高金額訂單核可 | approval |
| 帳號角色變更 | approval |

### 9.2 簽章流程（Part 11 11.200 兩組件）

```
使用者點「簽章」
  ↓
彈窗顯示：
  - 你正在簽：<記錄類型 + 摘要>
  - 意涵：<review / approval / responsibility / authorship>
  - 你的姓名：<從 M365 帶入>
  - 你的角色：<...>
  - 簽章時間：<server now, UTC + local>
  - 請輸入密碼以確認（密碼 = M365 密碼，透過 Entra resource owner password 流程驗證；若公司禁用 ROPC，改為「按下後跳出 Entra interactive auth」彈窗）
  ↓
按下「我已閱讀並同意」
  ↓
後端：
  1. 重認證使用者（Entra）
  2. 計算 hash：sha256(user_id + record_type + record_id + meaning + record_canonical_json + timestamp)
  3. INSERT e_signatures 表
  4. 將 e_sig.id 寫回原記錄（不可變更，只能新增 supersede 簽章）
  5. INSERT audit_log（hash chain）
  6. 顯示簽章證明（PDF stamp）
```

### 9.3 簽章呈現（Part 11 11.50）

每份 PDF（DHR、NCR、CAPA、客訴、quality doc）末尾必含：

```
─── Electronic Signatures ────────────────────────────────
[1] 王小明 (qa_manager) — Approval — 2026-04-30 14:32:11 UTC
    意涵：Final QC pass for order ORD-20260430-0001 lot L240430-A
    Signature ID: esig_01HXX...
─────────────────────────────────────────────────────────
```

### 9.4 簽章記錄資料模型

```sql
CREATE TABLE e_signatures (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  user_name_snapshot TEXT NOT NULL,    -- 簽章當時姓名（不隨後續改名變動，Part 11 11.70）
  user_role_snapshot TEXT NOT NULL,
  record_type TEXT NOT NULL,           -- 'order_stage' / 'ncr' / 'capa' / 'complaint' / 'document' / ...
  record_id TEXT NOT NULL,
  meaning TEXT NOT NULL,               -- 'review' / 'approval' / 'responsibility' / 'authorship'
  reason TEXT,                          -- 自由文字（QC unpassed reason 等）
  record_hash TEXT NOT NULL,           -- 簽章當時記錄 canonical JSON 的 sha256
  signed_at TIMESTAMP NOT NULL,
  reauth_method TEXT NOT NULL,         -- 'm365_password' / 'm365_mfa_step_up'
  ip_address TEXT,
  user_agent TEXT,
  prev_audit_hash TEXT,                -- audit chain 連結
  audit_id TEXT NOT NULL,              -- FK audit_log.id
  superseded_by TEXT,                   -- 後續更正簽章（不刪除原簽，新增 supersede）
  CONSTRAINT esig_no_delete CHECK (1=1) -- 應用層強制 no-DELETE
);
CREATE INDEX idx_esig_record ON e_signatures(record_type, record_id);
```

> 「不可變更」於 D1 沒有 row-level lock，靠 (a) 應用層只暴露 INSERT，(b) audit_log hash chain 偵測，(c) 定期 R2 immutable backup。

### 9.5 失效處理

- 員工離職：`users.disabled = true`，但歷史簽章保留（Part 11 11.300(d)：lost ID/password 後續仍 valid 直到主動 invalidate）
- 簽章修正：不修改原簽章，新增 supersede 簽章並關聯
- 偽簽防範：每次簽章必經 SSO 重認證；若 30 秒內 5 次密碼錯，鎖帳號 15 分鐘並 page admin

---

## 10. 稽核軌跡（Hash-chained, Immutable）

### 10.1 資料模型

```sql
CREATE TABLE audit_log (
  id TEXT PRIMARY KEY,                 -- ULID
  seq INTEGER NOT NULL UNIQUE,         -- 嚴格遞增，全域單調
  ts TIMESTAMP NOT NULL,               -- server UTC
  actor_user_id TEXT,                  -- null 表系統
  actor_source TEXT NOT NULL,          -- 'web' / 'api' / 'mcp' / 'system' / 'cron'
  ip_address TEXT,
  user_agent TEXT,
  action TEXT NOT NULL,                -- 'order.created' / 'esig.signed' / 'ncr.disposed' / ...
  record_type TEXT,
  record_id TEXT,
  before_json TEXT,                    -- 變更前狀態（diff 用）
  after_json TEXT,                     -- 變更後狀態
  metadata_json TEXT,                  -- 額外資料
  prev_hash TEXT NOT NULL,             -- 上一筆 audit_log 的 row_hash
  row_hash TEXT NOT NULL               -- sha256(canonical(this_row 除 row_hash 外))
);
CREATE INDEX idx_audit_record ON audit_log(record_type, record_id, ts);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id, ts);
CREATE INDEX idx_audit_seq ON audit_log(seq);
```

### 10.2 hash-chain 演算法

```
canonical = JSON.stringify({
  seq, ts, actor_user_id, actor_source, ip_address, user_agent,
  action, record_type, record_id,
  before_json, after_json, metadata_json,
  prev_hash
}, sorted keys)
row_hash = sha256(canonical)
```

寫入時：
1. SELECT MAX(seq), row_hash FROM audit_log → prev_seq, prev_hash
2. seq = prev_seq + 1
3. 計算 row_hash
4. INSERT（D1 transaction）

驗證時（年度稽核）：
- 從 seq=1 起逐筆重新計算，與 stored row_hash 比對
- 若有任一筆不符 → 整鏈斷裂，列出斷點供調查

### 10.3 RFC 3161 timestamping（Phase 4）

- 每日 00:00 對最新 audit_log row_hash 取 RFC 3161 timestamp（free TSA：FreeTSA / DigiCert）
- 將 timestamp token 存入 `audit_timestamps` 表
- 提供「給定日期前的 audit log 不可被偽造」的外部第三方證據

### 10.4 不可變保護

- 應用層：route 層 INSERT only，無 UPDATE / DELETE 端點
- DB 層：D1 沒有 row-level immutability，但靠 trigger 阻擋（D1 支援基本 trigger）：

```sql
CREATE TRIGGER no_audit_update BEFORE UPDATE ON audit_log
BEGIN SELECT RAISE(ABORT, 'audit_log is immutable'); END;
CREATE TRIGGER no_audit_delete BEFORE DELETE ON audit_log
BEGIN SELECT RAISE(ABORT, 'audit_log is immutable'); END;
```

- 備援：每日 export 全 audit_log 至 R2 immutable bucket（write once policy）

### 10.5 保留期

- audit_log：**7 年**
- 每年舊紀錄歸檔至 R2 cold storage（`audit-archive/<year>/`），保留可查詢能力

---

## 11. QMS 模組

### 11.1 NCR（Nonconforming Product, 820.90）

```
建立 NCR
  - 來源：incoming inspection / in-process / final QC / customer complaint
  - 不合格描述、影響批次（lot_no list）、影響數量、影響機台
  - 立即措施（隔離 quarantine / 標記 hold）→ 自動將相關 order_items.acceptance_status = 'quarantine'
  ↓
QA 主管調查 + disposition（e-sig）
  - rework / scrap / use-as-is（需風險評估文件） / return-to-supplier
  - 影響 DHR（自動連結至受影響訂單的 DHR）
  ↓
（若需要）建立 CAPA
  ↓
結案
```

資料模型：`ncr (id, ncr_no, source, source_ref_id, description, affected_lots_json, affected_qty, severity, reported_by, reported_at, target_disposition_date, disposition, disposition_reason, disposed_by, disposed_at, esig_id, capa_id, status, closed_at)`

### 11.2 CAPA（820.100）

```
建立 CAPA
  - 來源：NCR / 客訴 / 內部稽核 / 管理審查
  - 問題描述、根本原因分析（5-Why or fishbone 附件）
  - Action plan（多筆 action_items：負責人、due date）
  - Owner: <user>
  ↓
執行 actions
  - 每筆 action 完成後標記 + 證據文件
  ↓
Effectiveness 評估（30 / 60 / 90 天後）
  - 自動排定 review reminder
  - QA 主管評估是否有效（e-sig）
  ↓
結案 / 重啟（無效則回到 root cause 分析）
```

資料模型：`capa (id, capa_no, source, source_ref_id, problem, root_cause, owner_id, target_close_date, status, closed_by, closed_at, effectiveness_verified, esig_id_close)` + `capa_actions (id, capa_id, description, owner_id, due_date, completed_at, evidence_doc_id)`

### 11.3 Customer Complaints（820.198）

```
登錄客訴
  - 來源：客戶來電 / Email / 業務轉達 / 上市後監督
  - 受影響產品（SKU + lot_no）
  - 嚴重度初判：death / serious_injury / malfunction_could_cause_serious_harm / other
  ↓
觸發法定通報計時器（若 severity ≥ malfunction_could_cause）
  - TFDA：30 日（嚴重不良反應 7 日）
  - FDA MDR (21 CFR 803)：30 日（death/serious injury）, 30 日（malfunction）
  ↓
QA 主管調查
  - 調查報告（描述、原因、是否需召回、是否需通報）
  - 連結 NCR / CAPA（若需要）
  ↓
通報判定（e-sig）
  - 是否通報 TFDA / FDA / 其他主管機關
  - 通報 reference number 紀錄
  ↓
回覆客戶 + 結案（e-sig）
```

資料模型：`complaints (id, complaint_no, customer_id, received_at, received_via, product_sku, lot_no, severity, description, investigation, root_cause, requires_recall, requires_mdr_report, mdr_reference, response_to_customer, response_at, closed_by, closed_at, esig_id_close)` + `complaint_attachments (...)`

### 11.4 Document Control（820.40 / Part 11 11.10(k)）

兩個 document 子系統，**不要混淆**：

| 子系統 | 對象 | 範例 |
|-------|------|------|
| **Order Documents** | 與某筆訂單綁定 | PO、Quote、BOM、Packing List、Invoice、Shipping Doc、DHR PDF |
| **Quality Documents** | 全域文件管制 | SOP、Work Instruction、表單範本、產品規格、訓練教材 |

Quality Documents 流程：
```
Draft（撰寫者）
  ↓
Review（reviewer e-sig）
  ↓
Approved（approver e-sig，必須 ≠ 撰寫者，SoD-4）
  ↓
Effective（生效日，可排程未來日）
  ↓
Superseded（新版發布，舊版自動 superseded，保留 7 年）
  ↓
Obsoleted（不再使用）
```

資料模型：`quality_documents (id, doc_no, title, doc_type, version, status, effective_from, effective_to, content_r2_key, created_by, reviewed_by, approved_by, esig_review_id, esig_approve_id, superseded_by, retention_until)` + `document_revisions (id, doc_id, version, change_summary, ...)`

### 11.5 Training Records（820.25）

```
- 每位員工有 training profile：completed_trainings[]
- Quality Document 標記為「require_training=true」+ trainees=[role list]
- 文件發布或 revision → 系統發 training 任務給對應 role 員工
- 員工點擊「我已閱讀並了解」→ 記錄 training_completion + 簽章
- 角色指派時驗證：所需 trainings 都完成才允許指派
```

資料模型：`trainings (id, doc_id, doc_version, ...)` + `training_assignments (id, training_id, user_id, assigned_at, completed_at, esig_id)`

---

## 12. 物流整合

延續 v0.1：黑貓（國內）+ 17track 聚合（國際 DHL/FedEx/UPS/EMS）。

QMS 變更：
- 物流事件納入 audit_log（透過 `actor_source='system'`）
- 出貨後若收件確認（`event_code=delivered`），自動觸發 release stage 完成事件並寫入 DHR

---

## 13. 文件管理（Order Docs + Quality Docs + DHR）

### 13.1 Order Documents

延續 v0.1：6 種類別（PO / Quote / BOM / Packing List / Invoice / Shipping Doc）+ R2 + presigned upload + 系統產生 PDF（Quote / Invoice）。

新增：
- DHR 自動產生（`generate_dhr(order_id)` 拼合所有 stage records、acceptance、簽章、UDI、lot info）
- 病毒掃描：Phase 4 接入（ClamAV worker proxy）

### 13.2 Quality Documents

獨立模組，非綁定訂單。詳見 §11.4。

### 13.3 R2 命名與 immutability

```
order-docs/<order_id>/<doc_type>/v<n>-<ulid>-<filename>
quality-docs/<doc_no>/v<version>/<filename>
audit-archive/<year>/<month>/audit-log-<batch>.jsonl.gz
dhr/<order_id>/dhr-<order_no>-v<n>.pdf
ncr-attachments/<ncr_id>/<filename>
capa-attachments/<capa_id>/<filename>
complaint-attachments/<complaint_id>/<filename>
```

R2 bucket lifecycle：
- 一般 R2：可覆寫
- `r2-immutable` bucket（單獨）：write once（透過 worker policy 拒絕 PUT 已存在的 key）
- audit-archive 寫入 `r2-immutable`

---

## 14. 報表

### 14.1 基本營運報表（Phase 1）

- SLA breach 統計（按 stage / 月）
- 月成交量、月營業額
- AR aging（accountant 用）

### 14.2 QMS 報表（Phase 3-4）

- **NCR trend**：按來源 / 嚴重度 / 月，閉環率
- **CAPA trend**：開立數 / 結案數 / 平均結案天數 / effectiveness pass rate
- **客訴 trend**：按嚴重度 / 月，平均回應天數，MDR 通報統計
- **訓練覆蓋**：每份 quality doc 的指派 vs 完成率
- **角色變更紀錄**：上季所有 role assignment changes
- **Audit log integrity report**：hash-chain 驗證結果

### 14.3 年度 QMS 稽核報表（Phase 4）

一鍵彙整以下章節，輸出 PDF + Excel：
1. 管理審查 KPI（ISO 13485 §5.6 inputs）
2. 客訴趨勢與 MDR 通報
3. NCR 與 CAPA 趨勢
4. 訓練完成率
5. 文件發布／更新紀錄
6. 角色變更與 access review 結果
7. Audit log hash-chain 完整性驗證
8. 軟體變更記錄（從 commit log 擷取）

---

## 15. 資料完整性 ALCOA+

每筆紀錄需滿足：

| 原則 | 系統實作 |
|------|---------|
| **A** Attributable（可歸屬） | 必填 `created_by` / `actor_user_id` |
| **L** Legible（可辨讀） | UTF-8 + 結構化 JSON + PDF 匯出 |
| **C** Contemporaneous（同步紀錄） | server timestamp，不允許客戶端時間 |
| **O** Original（原始） | 原始 record + audit_log before/after 都保存 |
| **A** Accurate（正確） | DB constraint + Zod schema 雙驗 |
| **C** Complete | 必填欄位限制 |
| **C** Consistent | Hash chain |
| **E** Enduring | 7 年保留 |
| **A** Available | 有時可恢復 + R2 backup |

---

## 16. 資料保留與備份

### 16.1 保留期

| 資料類型 | 保留期 | 政策依據 |
|---------|-------|---------|
| 訂單 / 訂單品項 | 7 年（從出貨日起） | TFDA / FDA |
| DHR | 7 年（或產品壽命 + 2 年，取較長者） | 820.184 |
| NCR / CAPA / 客訴 | 7 年 | 820.180 |
| Quality Documents | 7 年（從 superseded 起） | 820.40 |
| Training Records | 員工離職 + 7 年 | 820.25 |
| Audit Log | 7 年 | 820.180 / Part 11 |
| E-signatures | 同被簽章記錄保留期 | Part 11 11.70 |

過期資料：歸檔至 R2 cold + 從主 D1 移除（保留索引指向歸檔）。

### 16.2 備份政策

| 對象 | 頻率 | 目的地 | 加密 |
|------|------|-------|------|
| D1 全表 | 每日 02:00 | R2 `backups/<date>/` | AES-256（Workers 端） |
| R2 documents | Cloudflare 自動三副本 | 跨區複製到第二 bucket | R2 內建 |
| Audit log | 每日 export jsonl.gz | R2 immutable | 內建 |
| 設定（wrangler secrets） | 每月 export | 1Password vault（手動） | 1Password |

每季演練還原至 staging Worker（restore drill），記錄 RPO（< 24h）/ RTO（< 4h）。

---

## 17. 軟體驗證（Software Validation）

### 17.1 Validation Master Plan（VMP）

- 系統屬於「QMS-supporting software」（820.70(i) software validation 適用）
- Risk classification：medium-high（影響產品 release 決策）
- 驗證範圍：所有 Part 11 / 820 對應功能（見 §2 對照表）

### 17.2 IQ / OQ / PQ

- **IQ（Installation Qualification）**：部署環境、Cloudflare 配置、binding、密鑰存放、版本標記
- **OQ（Operational Qualification）**：每個模組的功能測試（自動化測試 + 手動 checklist）
- **PQ（Performance Qualification）**：典型業務情境端到端測試（接單 → 出貨 → DHR）

每次 major release 重跑 OQ／PQ；patch release 跑 regression + 影響範圍 OQ。

### 17.3 Traceability Matrix

每條 user requirement → design spec section → test case → test result。維護於 `docs/validation/traceability-matrix.csv`。

### 17.4 Change Control

- 任何 production deploy 需 PR + reviewer 簽核 + change record（`change_records` 表）
- High-risk change（schema / Part 11 機制）：admin + qa_manager 雙簽 + regression 測試報告
- Emergency hotfix：先修，72h 內補 change record + retro

### 17.5 Periodic Review

- 每季：OQ smoke test 重跑、access review、hash-chain integrity check
- 每年：完整 validation refresh、災難演練、滲透測試

---

## 18. 部署架構

### 18.1 Topology

```
[公司員工 / 內網]
   ↓
[Cloudflare Access — Zero Trust gate]
   identity provider: M365 Entra ID
   policy: require MFA + (Phase 5: device posture)
   ↓
[Pages: orderflow.internal.<company>.com]   ← 靜態前端
[Workers: api.orderflow.internal.<company>.com]  ← API
[Workers: mcp.orderflow.internal.<company>.com]  ← MCP server
   ↓
[D1: orderflow-prod]
[R2: orderflow-docs / orderflow-immutable]
[KV: orderflow-cache]
[Queues: orderflow-notifications]
```

- 公司不對外公開 URL；DNS 為內網專用 + Cloudflare Tunnel（若需遠端）
- 環境分離：`local` / `staging` / `prod`，各環境獨立 D1 + R2 + Cloudflare Access policy

### 18.2 Secrets

存於 Wrangler secrets：
- `M365_CLIENT_SECRET`
- `RESEND_API_KEY`
- `TEAMS_WEBHOOK_URL`
- `SLACK_WEBHOOK_URL`
- `VAPID_PRIVATE_KEY`
- `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY`
- `MCP_INTERNAL_TOKEN`
- `JWT_SIGNING_KEY` / `AUDIT_HASH_PEPPER`

### 18.3 Network controls

- TLS 1.3 only（Cloudflare 預設）
- HSTS preload
- CSP strict + nonce
- CORS：僅 internal subdomain
- Rate limiting：Cloudflare WAF + 應用層 token bucket

---

## 19. 目錄結構

```
orderflow/
├── package.json                   # bun workspace
├── tsconfig.base.json
├── docs/
│   ├── superpowers/
│   │   ├── specs/
│   │   └── plans/
│   ├── validation/
│   │   ├── vmp.md
│   │   ├── iq-checklist.md
│   │   ├── oq-tests.md
│   │   ├── pq-scenarios.md
│   │   └── traceability-matrix.csv
│   ├── sops/
│   │   ├── access-review.md
│   │   ├── change-control.md
│   │   ├── incident-response.md
│   │   ├── backup-restore.md
│   │   └── disaster-recovery.md
│   └── DEPLOYMENT.md
├── apps/
│   ├── web/                       # React + Vite + Tailwind PWA
│   ├── api/                       # Cloudflare Workers + Hono + Drizzle
│   └── mcp/                       # MCP server (internal Teams Copilot)
└── packages/
    ├── shared/                    # Zod schemas, types
    ├── qms/                       # NCR / CAPA / complaints / training logic
    └── crypto/                    # hash chain + e-sig helpers
```

---

## 20. 實作路線圖（5 phases）

| Phase | 範圍 | 估時 | 主要 deliverable |
|-------|------|------|----------------|
| **Phase 0** | 單一公司 Foundation：M365 SSO + Cloudflare Access + hash-chain audit + e-sig 框架 + RBAC + UI shell | 2-3 週 | 員工 SSO 進系統，看到空 shell，登入有 audit + 第一個簽章 demo |
| **Phase 1** | Order MVP + Lot/UDI/Serial + e-sig on critical stages + Documents + DHR + Alerts + Logistics + Kanban | 6-7 週 | 完整接單到出貨流程跑起來，DHR 可印 |
| **Phase 2** | Mobile PWA + UDI 條碼掃描 + 進階驗收（incoming/in-process/final QC）+ access review + 進階權限 | 4 週 | ops 可手機掃 UDI、品保可結算 release |
| **Phase 3** | QMS modules：NCR / CAPA / 客訴 / Quality Document Control / Training Records | 5 週 | 公司可在系統內處理不合格品、開 CAPA、登客訴 |
| **Phase 4** | 年度稽核報表 + MCP for Teams Copilot（內部）+ 軟體驗證 IQ/OQ/PQ + RFC 3161 timestamping + 災難演練 | 4 週 | 通過 ISO 13485 / 21 CFR 820 / Part 11 模擬稽核 |

合計約 21-23 週。

---

## 21. 變更紀錄

| 版本 | 日期 | 摘要 |
|------|------|------|
| v0.1 | 2026-04-30 | 初版：multi-tenant SaaS 定位 |
| v0.2 | 2026-04-30 | 重新定位：單一公司內部部署，加入 ISO 13485 + 21 CFR 820 + Part 11 完整合規範圍，新增 Lot/UDI/DHR/NCR/CAPA/客訴/文件管制/訓練記錄；audit_log hash-chain；electronic signature 框架；新增 Phase 4 |

---

## 22. 待解決事項（Open Items）

1. **公司 Entra tenant ID**：取得後填入 `M365_TENANT_ID` secret。
2. **法規外部委任顧問**：建議聘請 ISO 13485 / FDA 顧問做最終 gap analysis（spec 是 self-assessment，非正式 audit）。
3. **DMR 連結方式**：Class II DMR 由現有 PLM 系統管理，本系統需要 PLM 提供 read API 或 export 機制（Phase 1 規劃時對齊）。
4. **TFDA / FDA 通報實際流程**：目前是手動上傳通報文件（不在本系統內 e-submit）。系統僅追蹤 deadline 與通報狀態。
5. **Cloudflare Access licensing**：確認公司 Cloudflare 方案支援 Zero Trust（Free 50 users / Standard / Enterprise）。
6. **災難演練：場外備份**：是否需要將 R2 backup 再 mirror 至 AWS S3 / Google Cloud Storage 以避免單點供應商風險？建議列入 Phase 4。
