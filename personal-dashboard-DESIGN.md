# Personal Dashboard — 設計與維護規則

- **版本**：v3.0
- **日期**：2026-09-02
- **狀態**：P1/P2 完成，P3 起未動工
- **權威性**：本檔為 schema 與規則的唯一權威來源。Notion 是資料容器，本檔是結構定義。CSV 匯出無法還原 relation 與 rollup 設定，因此結構必須在此版控。

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

**已知的 Notion 弱點（接受但需緩解）**
1. hosted MCP 無 subtree OAuth scoping，以登入者完整權限行動 → 用 internal connection token 路徑緩解（§3）
2. 無文件化的 CAS / ETag / idempotency key → 用 OpID 欄位 + 序列化人工驅動緩解（§5、§6）
3. CSV 匯出不保留 relation 語意 → 用 GoalKey 文字欄緩解（§5）

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
- Capabilities 只勾 **Read content + Insert content**。**不勾 Update content** —— 從技術層面消滅「排程 job 覆蓋手動編輯」這個 failure mode。
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

---

## 5. Schema 定義（權威）

### Goals
| 欄位 | 型別 | 說明 |
|---|---|---|
| Name | title | |
| Key | text | 唯一 slug，人工指定，如 `g-etf-2026q4`。匯出後的 join key |
| Area | select | 見下方 Area 選項 |
| Horizon | select | `week` / `quarter` / `year` / `ongoing` |
| Status | select | `active` / `paused` / `done` / `dropped` |
| Metric | text | 怎麼量 |
| Target | text | 量到多少算達成 |
| NextReview | date | |

### Tasks
| 欄位 | 型別 | 說明 |
|---|---|---|
| Name | title | |
| Due | date | 提醒用，非行事曆真值 |
| Area | select | |
| Status | select | `todo` / `doing` / `blocked` / `done` |
| GoalKey | text | agent 寫這欄 |
| Goal | relation → Goals | 人工在 UI 連，agent 不碰 |
| Blocker | text | |
| OpID | text | |
| Source | select | |

### Log（已建立，schema 已驗證吻合）
| 欄位 | 型別 | 說明 |
|---|---|---|
| Name | title | |
| Date | date | |
| Area | select | |
| GoalKey | text | |
| Delta | text | |
| Evidence | url | 指向 GitHub / Drive，不上傳檔案 |
| OpID | text | |
| Source | select | |

> Log 目前尚未加 `Goal(relation)`。P3 建 Goals 後補上。

### 共用 select 選項
```
Area   : quanta | grad-school | etf | hardware | skills | fitness | career
Source : manual | claude | codex | grok | script
Horizon: week | quarter | year | ongoing
```

### 三個設計約束的理由

**1. Area 用 select 不用 relation 表**
agent 單表查詢即可看到 area，免第二次 fetch，免跨表 SQL（Free 方案不支援多源 SQL）。

**2. GoalKey(text) 與 Goal(relation) 並存**
官方文件：CSV 匯出後 relation 只剩 plain text URL，且 CSV 不能重新匯入重建 relation。relation 服務 UI 的 rollup；GoalKey 服務匯出與 agent。備份靠 GoalKey 還原拓樸。

**3. OpID(text) 取代 write broker**
```
格式：{agent}-{YYYYMMDDTHHMMSS}-{slug}
寫入前：先用 view 查同 OpID 是否已存在
```
誠實標註：**這不是原子操作，理論上仍有 TOCTOU race。** 在單人序列驅動下足夠，成本是一個欄位而非一台服務。若日後轉為無人值守，這欄正好是升級成 broker 時的現成 dedup key。

Broker 對本案是過度設計：需自架、需公開 HTTPS 供 Grok/ChatGPT、自身成為新 SPOF —— 引入的正是本案要避開的基礎設施。

---

## 6. View 定義

Agent 只准讀 view，不准跑 SQL。

| View | 資料源 | 過濾 | 排序 | 狀態 |
|---|---|---|---|---|
| `Agent: Log 14d` | Log | `Date >= today-14` | Date desc | ✅ 已建 |
| `Agent: Open Tasks` | Tasks | `Status != done` | Due asc | ⬜ |
| `Agent: Active Goals` | Goals | `Status = active` | — | ⬜ |
| `Agent: OpID Lookup` | Log | 無過濾 | Date desc | ⬜ |
| `Human: Dashboard` | page | linked views 集合 | — | ⬜ |

**維護注意**：`Agent: Log 14d` 目前是 hardcoded 日期 `2026-08-19`，不是滾動視窗。需定期更新，或改用相對日期過濾。**這是已知技術債。**

---

## 7. Agent 寫入規約

貼入三家 agent 的 system prompt / project instructions。

```
Notion write protocol — personal dashboard

SCOPE
- Databases: Goals, Tasks, Log. Nothing else in the workspace.

WRITE
- Create rows, or update a single property of an existing row. Nothing else.
- Do not write page bodies. notion-update-page's update_content is a targeted
  search-and-replace (fails safely if old_str is absent), but replace_content
  on the same tool replaces the whole page and has a history of deleting child
  content. Stay off the tool entirely.
- NEVER use erase_content. It is irreversible via API.
- Log is append-only. Do not modify existing Log rows.
- Goals rows are human-owned. You may append to Log or add a comment to
  propose a change, but must not write Goals.Status or Goals.Target.
- Write GoalKey (text). Do not write the Goal relation.
- Set OpID and Source on every row you create.
  OpID format: {agent}-{YYYYMMDDTHHMMSS}-{slug}
  Check for an existing row with the same OpID before creating.
- Deleting, archiving, or moving anything requires asking the user first.

READ
- Use saved views: query_data_sources with mode "view".
- Do not run SQL. Single-source SQL is metered on this plan; view mode is not.

RATE LIMITS
- 180 req/min (3 req/s) per user across all MCP tool calls.
- notion-search separately capped at 30 req/min.
- A workspace-wide limit shared across all connections also applies
  (value not published by Notion).
- Do not run parallel searches.
```

規約是**軟約束**。MCP 層面 `update_page` 是 available，擋不住，只能靠 prompt + 事後用 Source/OpID 稽核。排程路徑因為不勾 Update content，才是硬約束。

---

## 8. 限制與已驗事實

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

## 9. 備份與復原

| 層級 | 機制 | 涵蓋 | 限制 |
|---|---|---|---|
| L1 | Notion 版本歷史 | 7 天 | 自助，Free 上限 |
| L2 | Notion support snapshot | 30 天 | 需開 ticket，vendor-dependent |
| L3 | **每週 CSV export → GitHub repo** | 無限 | 需手動或排程 |
| L4 | **本檔** | 結構定義 | 版控在 repo |

L3 + L4 缺一不可：CSV 救資料，本檔救結構。單有 CSV 無法重建 relation 與 rollup。

三個 agent 都有寫入權，7 天自助歷史不足以涵蓋 → **L3 非可選項。**

---

## 10. 建置進度

```
✅ P1  Log 表 + Agent: Log 14d view              (2026-09-02, by codex)
✅ P2  create → view mode read 驗證通過           (2026-09-02)
⬜ P3  建 Dashboard page；搬 Log 進去；
       建 Goals + Tasks + 3 個 views；Log 補 Goal relation
⬜ P4  建 internal connection，只 connect Dashboard 子樹
       capabilities: Read + Insert only
⬜ P5  排程 script：append Log + weekly CSV export
```

### 待處理
- `__noop__` database（頂層，僅 Name 欄位，無內容）—— 疑為 `create_database` 探測殘留，待確認後刪除
- `Agent: Log 14d` 的 hardcoded 日期需改為滾動視窗

### 未解問題
- ChatGPT 方案別未確認（影響 web 端是否可寫）
- Free 方案 SQL 配額實際數字未知（架構已避開，僅供參考）
- 若日後轉為無人值守大量寫入，需重新評估 Airtable（PAT 可 scope 到單一 base、Web API 有 `performUpsert`）

---

## 11. 變更紀錄

| 版本 | 日期 | 變更 |
|---|---|---|
| v1 | 2026-09-02 | 初版：4 表 + relation + agent 跨表 rollup |
| v2 | 2026-09-02 | 依 Free 方案實測收斂為 3 表；Area 改 select；agent 讀 view 不跑 SQL |
| v3 | 2026-09-02 | 加 GoalKey 與 OpID；否決 write broker；確立雙路徑存取；修正 `update_page` 語意描述；撤回「Notion 唯一符合」與「Grok connectors 限付費」兩項錯誤陳述 |
