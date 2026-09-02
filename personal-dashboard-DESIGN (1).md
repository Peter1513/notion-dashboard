# Personal Dashboard — 設計決策紀錄

- **版本**：v3.1
- **日期**：2026-09-02
- **狀態**：P1/P2 完成，P3 起未動工

---

## 0. 文件關係

本專案有三份文件，職責不同：

| 文件 | 性質 | 讀者 | 內容 |
|---|---|---|---|
| **README** | 操作性 | 人 / Agent | 怎麼 deploy、boot、operate、verify、troubleshoot |
| **OPERATIONS** | 規範性 | Agent | schema、view、寫入規約、復原範圍 —— agent 必須遵循 |
| **DESIGN**（本檔） | 說明性 | 人 | 為什麼這樣決定、否決了什麼、哪些事實已驗證 |

**規則衝突時以 OPERATIONS 為準。** README 說明怎麼做，本檔解釋為什麼，兩者皆不定義規則。Agent 不應從本檔推導行為。

權威副本在 git repo。Notion 頁面是方便閱讀的鏡像，可能落後。

---

## 1. 目的與範圍

### 目的
單一個人 dashboard：記錄 goals、progress、tasks，供人閱讀，並可被多個 AI agent 讀寫。

### 硬需求
| # | 需求 |
|---|---|
| R1 | 雲端；web + 官方 mobile app |
| R2 | 可被 Claude、Codex/ChatGPT、Grok 存取 |
| R3 | 成熟穩定，低營運風險 |

### Non-scope
- 不做團隊協作
- 不做行事曆的雙向同步
- 不自建 write broker 或任何常駐服務
- 不做無人值守的大量自動寫入（見 §3 存取模式 B）

---

## 2. 選型決策

**結論：Notion。**

| 候選 | R1 | R2 | R3 | 判定 |
|---|---|---|---|---|
| **Notion** | PASS | PASS | PASS | **採用** |
| Linear | PASS | PASS | PASS | 否決 — issue tracker 資料模型，無 goal/journal 語意 |
| Airtable | PASS | PASS | 存疑 | 否決 — 官方 MCP 文件自帶 tool 行為可能變動警告 |
| Coda | PASS | PASS | 存疑 | 否決 — MCP 官方標 beta |
| Google Workspace | PASS | PASS | 存疑 | 否決 — MCP 官方標 Developer Preview；dashboard UI 差 |
| AFFiNE | PASS | 部分 | 存疑 | 否決 — write tools 仍在 rollout |
| Anytype | PASS | FAIL | — | 否決 — 官方 MCP 為 stdio，Grok 需 public HTTPS |
| Capacities | PASS | PASS | 存疑 | 否決 — 需 Pro；生態規模小 |
| Obsidian | FAIL | PASS | PASS | 否決 — 無官方 web app |

**修正紀錄**：初版曾稱「Notion 是唯一符合三條件者」。此說法過強 —— Linear 同樣三項皆過。Notion 勝出的實際理由是資料模型契合度，不是唯一性。

### 已知的 Notion 弱點（接受但需緩解）

| # | 弱點 | 緩解 | 定義於 |
|---|---|---|---|
| 1 | hosted MCP 無 subtree OAuth scoping，以登入者完整權限行動 | internal connection token 路徑 | 本檔 §3 |
| 2 | 無文件化的 CAS / ETag / idempotency key | OpID 欄位 + 序列化人工驅動 | OPERATIONS §4 |
| 3 | CSV 匯出不保留 relation 語意 | GoalKey 文字欄 | OPERATIONS §2 |

---

## 3. 存取架構

存取模式 = **B（互動為主 + 少量排程）**。

```
                   ┌─ Claude  (claude.ai / Claude Code)
互動路徑 ──────────┼─ Codex   (CLI, ~/.codex/config.toml)
mcp.notion.com/mcp └─ Grok    (Connector Catalog)
   OAuth，你本人身分，完整權限
                                              ──> Notion
排程路徑 ──────────── 你的 script (pasoko / GitHub Actions)
Notion REST API        internal connection token
   最小權限
```

### 互動路徑
| 客戶端 | 設定 | 備註 |
|---|---|---|
| Claude Code | `claude mcp add --transport http notion https://mcp.notion.com/mcp` 後 `/mcp` 走 OAuth | 官方文件路徑 |
| Codex CLI | `~/.codex/config.toml` 加 `[mcp_servers.notion]` / `url = "https://mcp.notion.com/mcp"`，再 `codex mcp login notion` | local 配置，不受 ChatGPT Developer Mode 方案梯度限制 |
| ChatGPT web | 用 directory 的 **Notion 官方 app**（Developer 標為 Notion，含 Write） | **不要走 Developer Mode custom MCP** — Pro 只能 read/fetch，Plus 無此路徑 |
| Grok | Connector Catalog 的 Notion（第三方 OAuth，非 xAI built-in） | 出事找 Notion 不是找 xAI |

### 排程路徑
- 用 **internal connection**，不用 PAT。PAT 以建立者身分行動、繼承完整權限，等於把 hosted MCP 的問題原樣搬過來。
- Capabilities 只勾 **Read content + Insert content**。**不勾 Update content** —— 從技術層面消滅「排程 job 覆蓋手動編輯」這個 failure mode。這是整套設計中唯一的硬約束。
- 頁面必須手動 connect：頁面 `•••` → Add connections → 選該 connection。未 connect 的頁面 API 一律回錯。
- Token 存環境變數，不進原始碼、不進版控。

### 行事曆
Google Calendar 為唯一真值。Notion 的 `Tasks.Due` 只是提醒欄，不做雙向同步。

Notion Calendar（web/Mac/Win/iOS/Android，入口 `calendar.notion.so`）可當前端 UI 讀 GCal，純視覺選擇，不改架構。

---

## 4. Workspace 結構

```
Peter Wang's Space (Free)
└─ Dashboard                  page      ← 待建，token 的 connect 根節點
   ├─ Goals                   database  ← 待建
   ├─ Tasks                   database  ← 待建
   └─ Log                     database  ← 已存在，待搬入
```

母 page 是必要的，不是美觀考量：沒有它，排程路徑的最小權限就得逐張表 connect，且新增表時容易漏。

Schema 定義見 **OPERATIONS §2**。

---

## 5. 三個設計約束的理由

實際規則在 OPERATIONS，此處只記錄為何如此。

### 5.1 Area 用 select 不用 relation 表

agent 單表查詢即可看到 area，免第二次 fetch，免跨表 SQL（Free 方案不支援多源 SQL）。代價是 area 沒有自己的 metadata 頁 —— 接受，因為 area 只是分類標籤，不是實體。

### 5.2 GoalKey(text) 與 Goal(relation) 並存

官方文件：CSV 匯出後 relation 只剩 plain text URL，且 CSV 不能重新匯入重建 relation。

- `relation` 服務 UI 的 rollup
- `GoalKey` 服務匯出與 agent

備份靠 GoalKey 還原拓樸。這是刻意的資料冗餘，不是設計疏漏。

### 5.3 OpID 取代 write broker

Broker 對本案是過度設計：需自架、需公開 HTTPS 供 Grok/ChatGPT、自身成為新 SPOF —— 引入的正是本案要避開的基礎設施。

OpID 方案的誠實評價：**不是原子操作，理論上仍有 TOCTOU race。** 在單人序列驅動下足夠，成本是一個欄位而非一台服務。若日後轉為無人值守，這欄正好是升級成 broker 時的現成 dedup key。

---

## 6. 限制與已驗事實

### 已由本人實測驗證

| 項目 | 結果 | 方法 |
|---|---|---|
| Workspace | `Peter Wang's Space`，ID `754f2ec5-7392-818d-8cfa-000384801235`，Free | `fetch self` |
| `query_data_sources` view mode | **Free 可用，無 tool 配額** | 實際讀 `Agent: Log 14d` 成功 |
| `query_data_sources` SQL 單源 | `available_with_limit`（配額數字未公開） | `fetch self` tool access map |
| `query_multiple_data_sources` | `full_version_required`（Business + Notion AI） | 同上 |
| `query_meeting_notes` | `plan_required` | 同上 |
| create/update page、create_database、create_view、move_pages、comments | 全部 `available` | 同上 |
| hosted MCP 非互動授權 | **不支援**。官方 FAQ：目前必須完成 OAuth flow，非互動授權開發中 | fetch 官方文件 |
| internal connection | 靜態 token；需 workspace owner；頁面須手動 connect，否則 API 回錯 | 官方 authorization doc |

### 引用他處、本人未逐一驗證

- Free 方案：5MB 單檔上傳、7 天版本歷史、10 guests
- Notion 可從過去 30 天 snapshot 協助恢復誤刪（vendor-dependent，非自助）
- 無 scheduled automatic workspace backup
- rate limits：180 req/min、3 req/s、search 30 req/min
- `is_locked` 只擋 UI 不擋 API；`erase_content` 不可經 API 還原
- 2026-01-15 changelog：`replace_content` 曾致 child content 被刪
- CSV 匯出 relation → plain text URL，不能重匯入重建
- ChatGPT Developer Mode 方案梯度；Grok connectors 對所有使用者開放

**若要拿上述任一條做不可逆決策，先補查。**

---

## 7. 建置進度

```
✅ P1  Log 表 + Agent: Log 14d view              (2026-09-02, by codex)
✅ P2  create → view mode read 驗證通過           (2026-09-02)
⬜ P3  建 Dashboard page；搬 Log 進去；
       建 Goals + Tasks + 3 個 views；Log 補 Goal relation
⬜ P4  建 internal connection，只 connect Dashboard 子樹
       capabilities: Read + Insert only
⬜ P5  排程 script：append Log + weekly CSV export
```

每完成一階段，同步更新 OPERATIONS 的 §2 / §3 對應狀態。**OPERATIONS 先改，Notion 後動** —— 文件領先實作，不是反過來。

### 待處理
- `__noop__` database（頂層，僅 Name 欄位，無內容）—— 疑為 `create_database` 探測殘留，待確認後刪除
- `Agent: Log 14d` 的 hardcoded 日期需改為滾動視窗

### 未解問題
- ChatGPT 方案別未確認（影響 web 端是否可寫）
- Free 方案 SQL 配額實際數字未知（架構已避開，僅供參考）
- 若日後轉為無人值守大量寫入，需重新評估 Airtable（PAT 可 scope 到單一 base、Web API 有 `performUpsert`）

---

## 8. 變更紀錄

| 版本 | 日期 | 變更 |
|---|---|---|
| v1 | 2026-09-02 | 初版：4 表 + relation + agent 跨表 rollup |
| v2 | 2026-09-02 | 依 Free 方案實測收斂為 3 表；Area 改 select；agent 讀 view 不跑 SQL |
| v3.0 | 2026-09-02 | 加 GoalKey 與 OpID；否決 write broker；確立雙路徑存取；修正 `update_page` 語意描述；撤回「Notion 唯一符合」與「Grok connectors 限付費」兩項錯誤陳述 |
| v3.1 | 2026-09-02 | 拆出 OPERATIONS v1.0（schema / view / 寫入規約 / 備份）；本檔改為純決策紀錄；修正 v3.0 章節編號重複（兩個「2.」）；加 §0 文件關係 |
