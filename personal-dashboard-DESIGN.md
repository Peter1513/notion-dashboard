# Personal Dashboard — 設計決策紀錄

- **版本**：v3.7
- **日期**：2026-09-09
- **狀態**：P1/P2/P3/P3.1 完成；P4 暫緩；P5 未開始

---

## 0. 文件關係

本專案有三份文件，職責不同：

| 文件 | 性質 | 讀者 | 內容 |
|---|---|---|---|
| **README** | 操作性 | 人 / Agent | 怎麼 deploy、boot、operate、verify、troubleshoot |
| **OPERATIONS** | 規範性 | Agent | schema、view、寫入規約、復原範圍 —— agent 必須遵循 |
| **DESIGN**（本檔） | 說明性 | 人 | 為什麼這樣決定、否決了什麼、哪些事實已驗證 |

**規則衝突時以 OPERATIONS 為準。** README 說明怎麼做，本檔解釋為什麼，兩者皆不定義規則。Agent 不應從本檔推導行為。

權威副本只存在本 GitHub repo。2026-09-03 起不再於 Notion 維護 DESIGN / OPERATIONS 文件鏡像；後續 agent 必須先讀 repo 內的文件，而不是搜尋 Notion 內的規則頁。branch 由使用者指定，不由文件釘死。

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
| 1 | hosted MCP 無 subtree OAuth scoping，以登入者完整權限行動 | internal connection token 路徑（P4，目前暫緩） | 本檔 §3 |
| 2 | 無文件化的 CAS / ETag / idempotency key | OpID 欄位 + 序列化人工驅動 | OPERATIONS §4 |
| 3 | CSV 匯出不保留 relation 語意 | GoalKey 文字欄 | OPERATIONS §2 |

---

## 3. 存取架構

目前實際啟用的是互動路徑；排程路徑仍是設計目標，但 P4/P5 已暫緩。

```text
                   ┌─ Claude  (claude.ai / Claude Code)
互動路徑 ──────────┼─ Codex   (CLI, ~/.codex/config.toml)
mcp.notion.com/mcp └─ Grok    (Connector Catalog)
   OAuth，你本人身分，完整權限
                                              ──> Notion

排程路徑（DEFERRED）
script / GitHub Actions / other trusted HTTPS client
        │
        ▼
Notion REST API
internal connection token
Read + Insert only / Update disabled
```

### 互動路徑
| 客戶端 | 設定 | 備註 |
|---|---|---|
| Claude Code | `claude mcp add --transport http notion https://mcp.notion.com/mcp` 後 `/mcp` 走 OAuth | 官方文件路徑 |
| Codex CLI | `~/.codex/config.toml` 加 `[mcp_servers.notion]` / `url = "https://mcp.notion.com/mcp"`，再 `codex mcp login notion` | local 配置，不受 ChatGPT Developer Mode 方案梯度限制 |
| ChatGPT web | 用 directory 的 **Notion 官方 app**（Developer 標為 Notion，含 Write） | **不要走 Developer Mode custom MCP** — Pro 只能 read/fetch，Plus 無此路徑 |
| Grok | Connector Catalog 的 Notion（第三方 OAuth，非 xAI built-in） | 出事找 Notion 不是找 xAI |

### 排程路徑（P4/P5，DEFERRED）
- 預定仍使用 **internal connection**，不用 PAT。
- 預定 capabilities：**Read content + Insert content**；**Update content 關閉**。
- 預定只將 connection 掛到 `Dashboard` subtree。
- Token 應存 secret manager / environment storage，不進原始碼、不進版控、不貼入聊天。
- Local Bash 不是必要執行環境；未來可由 PowerShell、GitHub Actions、其他可信主機或等價 HTTPS client 驗證與執行。
- 目前 connection/scheduler 尚未完成驗證，因此不能宣告上述 hard constraint 已實際部署。

### 行事曆
Google Calendar 為唯一真值。Notion 的 `Tasks.Due` 只是提醒欄，不做雙向同步。

Notion Calendar（web/Mac/Win/iOS/Android，入口 `calendar.notion.so`）可當前端 UI 讀 GCal，純視覺選擇，不改架構。

---

## 4. Workspace 結構

```text
Peter Wang's Space (Free)
└─ Dashboard                  page
   ├─ Goals                   database
   ├─ Tasks                   database
   └─ Log                     database
```

母 page 是必要的，不是美觀考量：沒有它，未來排程路徑的最小權限就得逐張表 connect，且新增表時容易漏。

Schema 定義見 **OPERATIONS §2**。

---

## 5. 三個設計約束的理由

實際規則在 OPERATIONS，此處只記錄為何如此。

### 5.1 Area 用 select 不用 relation 表

agent 單表查詢即可看到 area，免第二次 fetch，免跨表 SQL（Free 方案不支援多源 SQL）。代價是 area 沒有自己的 metadata 頁 —— 接受，因為 area 只是分類標籤，不是實體。

### 5.1a Area 收斂為四值（v3.5）

原七值（quanta / grad-school / etf / hardware / skills / fitness / career）在 2026-09-08 首次匯入 24 筆 Tasks 時暴露兩個問題：`skills` 成為 catch-all（7 筆，含 dashboard 本身與各軟體專案），`etf` 過窄無法容納信用卡與記帳。

審核結論：Area 應回答「時間花在人生哪一塊」，而非「屬於哪個專案」；專案粒度已由 GoalKey 承載。採 Dan Koe 四市場 Health / Wealth / Relationships / Happiness 作為固定集合，理由：可枚舉、跨年不變、每個舊值都能唯一對應。Tasks / Log 的 Area 由所屬 Goal 繼承，agent 不自行判斷（參考 Ultimate Brain 的 Task→Project→Area 單鏈設計）。

否決方案：
- Area 改 multi_select —— 按 Area 統計會重複計數、每筆決策變慢、實務上退化為「每筆都貼兩個」。
- 新增 `projects` 值 —— 層級錯置，等同把 PARA 的 Projects 塞進 Areas。
- 保留七值僅改名 —— 治標，catch-all 問題會在下一個名字重演。

代價：Wealth 會承載當前多數項目（quanta / career / grad-school / finance 全歸此），接受，因為它反映現階段實況。遷移時原 Log 6 筆的 Area 值隨舊 option 移除而清空，接受為一次性損失。

### 5.2 GoalKey(text) 與 Goal(relation) 並存

官方文件：CSV 匯出後 relation 只剩 plain text URL，且 CSV 不能重新匯入重建 relation。

- `relation` 服務 UI 的 rollup
- `GoalKey` 服務匯出與 agent

備份靠 GoalKey 還原拓樸。這是刻意的資料冗餘，不是設計疏漏。

### 5.2a Status 用 status 型不用 select（v3.7）

OPERATIONS v1.0–v1.6 一律把 Goals / Tasks 的 `Status` 定為 `select`；實際上 Peter 已於 2026-09-08 在 Notion UI 將兩表轉為原生 `status` 型（To-do / In progress / Complete 三分組）。2026-09-09 稽核後決定**文件遷就現況**，不改回 select。

理由：`status` 是 Notion 對任務狀態的原生型別，board 與 group view 直接可用；agent 端讀寫的仍只是 option 名稱，寫入協定完全不變，因此改文件的成本低於改資料。

否決方案：改回 `select`（可讓 live 與 v1.5 文件一致，但要放棄分組 UI，且是為了遷就一份本來就落後於實況的文件）。

實測到的代價，兩項都已發生：

1. `select → status` 轉型會**清空所有引用該欄的 view filter**。2026-09-09 稽核時 `Agent: Active Goals` 與 `Agent: Open Tasks` 的 `advancedFilter.filters` 皆為空陣列，兩個 view 因此回傳全表。
2. Notion 轉型時會自動插入預設值 `Not started` 與 `In progress`，且 `Tasks` 的 `doing` 被 `In progress` 取代而消失。

故 OPERATIONS v1.7 明列 group 歸屬、宣告兩個預設值非法、要求補回 `doing`，並在 commit 後於 Notion 重建兩個 view 的 filter。

流程註記：本條款於 2026-09-09 已定案並產出 v1.6 補丁，但該補丁未 commit；`v1.6` / `v3.6` 版號隨後被「移除 beta branch 指定」一案佔用。此處以 v1.7 / v3.7 重新發布，內容與原決議等價。教訓：版號在 commit 落地前不算被佔用，決議與 commit 之間不應留下未追蹤的空窗。

### 5.3 OpID 取代 write broker

Broker 對本案是過度設計：需自架、需公開 HTTPS 供 Grok/ChatGPT、自身成為新 SPOF —— 引入的正是本案要避開的基礎設施。

OpID 方案的誠實評價：**不是原子操作，理論上仍有 TOCTOU race。** 在單人序列驅動下足夠，成本是一個欄位而非一台服務。若日後轉為無人值守，這欄正好是升級成 broker 時的現成 dedup key。

---

## 6. 限制與已驗事實

### 已由本人實測驗證

| 項目 | 結果 | 方法 |
|---|---|---|
| Workspace | `Peter Wang's Space`，ID `754f2ec5-7392-818d-8cfa-000384801235`，Free | `fetch self` |
| `query_data_sources` view mode | **Free 可用，無 tool 配額** | 實際讀 `Agent: Log 2026-09` 與 future-month `Agent: Log 2026-10` 成功 |
| `query_data_sources` SQL 單源 | `available_with_limit`（配額數字未公開） | `fetch self` tool access map |
| `query_multiple_data_sources` | `full_version_required`（Business + Notion AI） | 同上 |
| `query_meeting_notes` | `plan_required` | 同上 |
| create/update page、create_database、create_view、move_pages、comments | 全部 `available` | 同上 |
| hosted MCP 非互動授權 | **不支援**。官方 FAQ：目前必須完成 OAuth flow，非互動授權開發中 | fetch 官方文件 |
| internal connection design | capability 可區分 Read / Insert / Update；page sharing 決定 content scope | 官方文件；尚未完成 P4 live verification |
| P3 workspace structure | `Dashboard` 下已有 `Goals`、`Tasks`、`Log`；`Log.Goal` 與 `Tasks.Goal` 都指向 `Goals` | 2026-09-02 MCP create/fetch/move verification |
| Agent views | `Agent: Log 2026-01`～`2026-12`、`Agent: Open Tasks`、`Agent: Active Goals`、`Agent: OpID Lookup` 皆存在 | 2026-09-03 MCP create/fetch/view-mode verification |
| P3/P3.1 re-check | Dashboard/Goals/Tasks/Log 與既有 views 仍符合 OPERATIONS schema | 2026-09-03 live fetch verification |

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

```text
✅ P1    Log 表 + initial agent Log view           (2026-09-02, by codex)
✅ P2    create → view mode read 驗證通過           (2026-09-02)
✅ P3    Dashboard + Goals + Tasks + Log；
         3 個新 agent views；Log.Goal relation       (2026-09-02)
✅ P3.1  以 2026-01～2026-12 月度 Log views
         取代 rolling 14-day view                    (2026-09-03)
⏸ P4    internal connection / Dashboard subtree
         Read + Insert only；Update disabled          DEFERRED
⬜ P5    排程 script：append Log + weekly CSV export  NOT STARTED
```

目前 core dashboard implementation 可視為完成；scheduled automation path 暫緩，不因曾開始研究 P4 UI 而視為部分完成。

### 待處理
- 若恢復 P4：依當時最新官方 Developer UI 建 connection、設定 capabilities、只 connect Dashboard、再做 scope/API verification。
- P4 驗證完成後才進 P5。
- 2027 年開始前，依同一個月度邊界規則建立 `Agent: Log 2027-01`～`2027-12`，並先更新 OPERATIONS 版本。

### 已清理
- `__noop__` database 已於 2026-09-02 手動刪除，後續 workspace search 已確認 active workspace 不再存在該 database。
- 舊 Notion DESIGN / rules 文件鏡像已於 2026-09-03 由 owner 手動移除；後續只使用 GitHub repo 內的文件。

### 未解問題
- ChatGPT 方案別未確認（影響 web 端是否可寫）
- Free 方案 SQL 配額實際數字未知（架構已避開，僅供參考）
- P4 scheduler execution environment 尚未決定；Local Bash 已明確不是必要前提
- 若日後轉為無人值守大量寫入，需重新評估 Airtable（PAT 可 scope 到單一 base、Web API 有 `performUpsert`）

---

## 8. 變更紀錄

| 版本 | 日期 | 變更 |
|---|---|---|
| v1 | 2026-09-02 | 初版：4 表 + relation + agent 跨表 rollup |
| v2 | 2026-09-02 | 依 Free 方案實測收斂為 3 表；Area 改 select；agent 讀 view 不跑 SQL |
| v3.0 | 2026-09-02 | 加 GoalKey 與 OpID；否決 write broker；確立雙路徑存取；修正 `update_page` 語意描述；撤回「Notion 唯一符合」與「Grok connectors 限付費」兩項錯誤陳述 |
| v3.1 | 2026-09-02 | 拆出 OPERATIONS v1.0（schema / view / 寫入規約 / 備份）；本檔改為純決策紀錄；修正 v3.0 章節編號重複（兩個「2.」）；加 §0 文件關係 |
| v3.2 | 2026-09-02 | P3 完成：Dashboard/Goals/Tasks/Log 結構、relations 與三個缺少的 agent views 已部署；清理 `__noop__` 待辦。 |
| v3.3 | 2026-09-03 | 將 `Agent: Log 14d` 改為 2026 全年十二個月度 views；驗證 future-month view 可建立並以 view mode 回傳空陣列。 |
| v3.4 | 2026-09-03 | P4 明確標記為 deferred、P5 未開始；停止維護 Notion 文件鏡像，指定 GitHub `beta` 為唯一文件來源；記錄 Local Bash 非 P4/P5 必要執行環境。 |
| v3.5 | 2026-09-09 | Area 由七值收斂為 Health / Wealth / Relationships / Happiness；新增 §5.1a 記錄理由與否決方案；對應 OPERATIONS v1.5。 |
| v3.6 | 2026-09-09 | 移除文件內對 `beta` branch 的指定；文件來源改為「本 repo」，branch 由使用者決定。取代 v3.4 的 branch 條款；不維護 Notion 鏡像的決定不變。對應 OPERATIONS v1.6。 |
| v3.7 | 2026-09-09 | Status 改採 Notion `status` 型並記錄轉型副作用（view filter 被清空、插入預設值、`doing` 遺失）；新增 §5.2a；補上原定 v3.6 但未 commit 的決議。對應 OPERATIONS v1.7。 |