# personal-dashboard

## Index

- [Intro](#intro)
- [Requirements](#requirements)
- [Deploy](#deploy)
- [Boot](#boot)
- [Operation Guide](#operation-guide)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)

## Intro

`personal-dashboard` 是一組 Notion databases 加上治理文件，用來記錄個人 goals、tasks 與進度 log。

主要用途：

- 在 web 與手機上閱讀與更新個人進度。
- 讓 Claude、Codex、Grok 三個 agent 以受控方式讀寫同一份資料。
- 用文件而非服務來約束多 agent 寫入，避免自建 broker。

解決的問題：個人紀錄散落各處，且多個 AI agent 各自寫入時無法追溯來源、無法還原誤寫。

> 專案文件只以 GitHub `beta` branch 為來源。Notion 內不維護 DESIGN / OPERATIONS 鏡像；操作前先讀指定 branch 的文件。

## Requirements

Known-good environment:

| Item | Value |
|---|---|
| OS | Ubuntu 24.04 / WSL2 |
| Runtime | Notion Free workspace（雲端，無本機 runtime） |
| Shell | GNU Bash 5.x |
| Primary shell | `bash` |
| Main tool | Notion hosted MCP `https://mcp.notion.com/mcp` |

Required accounts:

```text
Notion workspace (owner 權限，建 internal connection 需要)
Claude / Codex / Grok 任一以上
GitHub repo (存放本專案文件與 CSV 備份)
```

Required files:

```text
README.md
personal-dashboard-OPERATIONS.md
personal-dashboard-DESIGN.md
```

Required environment variables（僅排程路徑需要，P5 前不需設定）:

| Variable | Meaning |
|---|---|
| `NOTION_TOKEN` | internal connection 的 installation access token |

Check environment:

```bash
claude --version
codex --version
git --version
```

Expected result:

```text
各工具版本字串；未安裝者顯示 command not found
```

未安裝的工具不影響其他路徑 —— 三家 agent 是可選的，至少接上一家即可運作。

## Deploy

此段說明從空 workspace 到可用狀態。

### 1. 建立 Notion 結構

P3 已完成。目前 workspace 結構：

```text
Peter Wang's Space
└─ Dashboard                  page
   ├─ Goals                   database
   ├─ Tasks                   database
   └─ Log                     database
```

Schema 欄位定義見 `personal-dashboard-OPERATIONS.md` §2。建表或改 schema 時以該檔為準，不要憑記憶填欄位。

目前已存在的 agent views：

```text
Agent: Log 2026-01 ... Agent: Log 2026-12
Agent: Open Tasks
Agent: Active Goals
Agent: OpID Lookup
```

月度 Log view 採固定日曆月邊界：`Date >= 當月第一天` 且 `Date < 下月第一天`。未來月份可以預先建立，尚無 matching row 時正常回傳空陣列。

### 2. 接上互動路徑

Claude Code:

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

Expected result:

```text
server 加入設定檔；下一步在 Claude Code 內執行 /mcp 完成 OAuth
```

Codex CLI —— 編輯 `~/.codex/config.toml`：

```toml
[mcp_servers.notion]
url = "https://mcp.notion.com/mcp"
```

登入：

```bash
codex mcp login notion
```

Expected result:

```text
瀏覽器開啟 Notion OAuth 頁；授權後 CLI 顯示連線成功
```

ChatGPT web —— GUI-operation-pseudocode:

```text
1. Open ChatGPT
2. Go to Settings > Apps & Connectors
3. Find "Notion" in the directory (Developer shown as Notion)
4. Click Connect and complete the Notion OAuth flow
5. Confirm the connector appears as Connected
```

不要走 Developer Mode custom MCP：Pro 只能 read/fetch，Plus 無此路徑。

Grok —— GUI-operation-pseudocode:

```text
1. Open grok.com/connectors
2. Locate Notion in the Connector catalog
3. Click Connect and complete the Notion OAuth flow
4. Confirm Notion appears in the connector list
```

### 3. 建立排程路徑 token — P4（DEFERRED）

P4 目前暫緩，不視為完成；P5 亦不得因此視為已啟動。以下保留目前官方 UI 路徑，供未來恢復 P4 時使用。

GUI-operation-pseudocode:

```text
1. Open Notion Web/Desktop
2. Settings > Developer > enable developer features
3. Return to the main sidebar > Developer > Connections
4. Click + New connection
5. Bind the connection to Peter Wang's Space
6. In Capabilities, enable ONLY "Read content" and "Insert content"
7. Leave "Update content" disabled
8. Keep user-information access disabled unless a future requirement explicitly needs it
9. Retrieve/copy the installation access token; never paste it into chat or commit it to git
10. Open the Dashboard page
11. Click ••• > Add connections > select the new connection
12. Do not separately connect unrelated workspace pages
```

Scope target：

```text
Dashboard
├─ Goals
├─ Tasks
└─ Log
```

P4 的目的不是讓 connection 自己執行工作，而是建立 P5 未來 scheduler 的 service identity：token 驗證身分、Capabilities 限制「能做什麼」、Dashboard sharing 限制「能碰哪裡」。

本機 Bash 不是 P4/P5 的必要條件。未來 REST verification / scheduler 可改由 PowerShell、GitHub Actions、其他可信主機或等價 HTTPS client 執行；credential 必須使用 secret/environment storage，不進 repo。

### Deploy Notes

```text
Free 方案單檔上傳上限 5 MB。Evidence 欄只放 URL，不上傳檔案。
若恢復 P4，不勾 Update content 是排程路徑的核心硬約束。
```

## Boot

本專案沒有長期執行的 process。Notion 是託管服務，無須啟動或停止。

以下為等價的連線檢查指令。

### Status — 互動路徑

在 Claude Code 內：

```text
/mcp
```

Expected result:

```text
notion 顯示為已連線
```

### Status — 排程路徑

目前 **DEFERRED / NOT VERIFIED**。恢復 P4 後，應以當時 Notion 官方最新 API version 與實際執行環境建立唯讀 smoke test；不要把下列舊版 Bash/curl 範例視為已驗證基準。

## Operation Guide

### Common Workflow

```text
1. Fetch README / OPERATIONS / DESIGN from GitHub beta
2. 開啟任一 agent，確認 Notion 已連線
3. 依需求月份讀 Agent: Log YYYY-MM，並讀 Agent: Open Tasks / Agent: Active Goals
4. 寫入 Log 或 Tasks，每列帶 OpID 與 Source
5. Weekly CSV export remains planned under P5, not active yet
```

### Common Commands

Agent 讀取一律走 view mode。以 Claude Code 為例，指示 agent：

```text
Read the exact "Agent: Log YYYY-MM" view for the requested month via query_data_sources with mode "view".
For a range spanning months, read each monthly view separately and combine the results.
Do not run SQL.
```

Expected result:

```text
回傳指定月份的列；未來月份或空月份回傳空陣列，不消耗 SQL 配額
```

### Important Options

| Item | Meaning | Value |
|---|---|---|
| `mode` | 讀取方式 | 固定 `view`；不使用 `sql` |
| `Source` | 寫入來源標記 | `manual` / `claude` / `codex` / `grok` / `script` |
| `OpID` | 去重鍵 | `{agent}-{YYYYMMDDTHHMMSS}-{slug}` |

### Config Edit Flow

Schema 或 view 變更一律文件先行：

```text
1. Edit personal-dashboard-OPERATIONS.md on beta, bump its version
2. Commit to beta
3. Apply the change in Notion
4. Re-run the smoke test in Verification
5. If Notion and OPERATIONS disagree, OPERATIONS wins; fix Notion
```

Notion 不再保存 DESIGN / OPERATIONS 文件鏡像，因此不存在「同步 Notion 文件頁」這一步。

### Logs

本專案的 log 就是 `Log` database。無本機 log 檔。

稽核誤寫時的查法：

```text
1. Open the Log database in Notion
2. Filter by Source to isolate one agent
3. Sort by Date descending
4. Cross-check OpID for duplicates
```

## Verification

### Smoke Test

確認連線與身分：

```text
Call notion-fetch with id "self".
```

Expected result:

```text
回傳 workspace name、workspace ID、user、以及 current_tool_access 對照表
```

確認 view mode 可讀：

```text
Call notion-query-data-sources with mode "view" on each agent view.
```

Expected result:

```text
每個 view 回傳列陣列；空表時回傳空陣列而非錯誤
```

### Manual Verification

```text
1. Open Notion on mobile
2. Navigate to Dashboard
3. Confirm Goals / Tasks / Log all appear under Dashboard
4. Confirm agent views are visible on their corresponding databases
```

### Evidence Format

```text
Command:
<tool call>

Exit code / status:
<ok | error code>

Output summary:
<key fields>

Result:
PASS / FAIL
```

### Observed Results

截至 2026-09-03：

```text
P1: PASS
P2: PASS
P3: PASS
P3.1: PASS
P4: DEFERRED / NOT VERIFIED
P5: NOT STARTED
```

P3/P3.1 evidence summary：

```text
Dashboard page created
Goals database created with OPERATIONS schema
Tasks database created with OPERATIONS schema
Log moved under Dashboard
Log.Goal -> Goals relation created
Tasks.Goal -> Goals relation created
Agent: Open Tasks created
Agent: Active Goals created
Agent: OpID Lookup created
Agent: Log 14d replaced by Agent: Log 2026-01 through Agent: Log 2026-12
2026-09-03 re-fetch confirmed Dashboard/Goals/Tasks/Log and views remain structurally consistent
```

## Troubleshooting

### Notion tools disappear mid-session

Symptom:

```text
Tool 'Notion:notion-search' not found
```

Diagnosis:

```text
Re-run tool discovery with a Notion-specific query.
If the result lists only Slack / Gmail / Drive / Calendar, the Notion tools were evicted.
```

Likely cause:

```text
同一 session 內載入多個 connector，Notion 的 tool 定義被擠出工具表。
非授權失效 —— 授權失效會回 auth 錯誤，不是 not found。
```

Fix:

```text
1. Re-run tool discovery with a Notion-specific query
2. If still absent, start a new conversation
3. If a new conversation still fails, reconnect Notion in the client's connector settings
```

Verify:

```text
Call notion-fetch with id "self".
```

Expected result:

```text
回傳 workspace 身分，代表工具已恢復
```

### query_multiple_data_sources fails

Symptom:

```text
full_version_required
```

Diagnosis:

```text
Call notion-fetch with id "self" and read current_tool_access.
```

Likely cause:

```text
跨 data source 的 SQL 需 Business 方案加 Notion AI。Free 方案不具備。
```

Fix:

```text
分別讀取各個 view，在 agent 端自行 join。不要嘗試單次跨表查詢。
```

Verify:

```text
Call notion-query-data-sources with mode "view" on each view separately.
```

Expected result:

```text
各自回傳結果，總和涵蓋所需資料
```

### Monthly Log view returns unexpected rows

Symptom:

```text
Agent: Log YYYY-MM 回傳其他月份的列，或應有紀錄卻為空
```

Diagnosis:

```text
Fetch the Log database and inspect the selected view's Date filters.
```

Likely cause:

```text
讀錯月份 view，或 month boundary 未採「當月第一天（含）至下月第一天（不含）」。
```

Fix:

```text
1. Confirm the requested calendar month
2. Open the exact "Agent: Log YYYY-MM" view
3. Set Date >= YYYY-MM-01
4. Set Date < next-month-01
5. Keep Date sorted descending
```

Verify:

```text
Call notion-query-data-sources with mode "view" on that monthly view.
```

Expected result:

```text
只回傳該日曆月的列；尚無紀錄的月份回傳空陣列
```

## Project Structure

```text
personal-dashboard/
├── README.md
├── personal-dashboard-OPERATIONS.md
└── personal-dashboard-DESIGN.md
```

Main files:

- `README.md`: deploy、boot、operation、verification、troubleshooting 說明。
- `personal-dashboard-OPERATIONS.md`: 規範性文件。schema、view、agent 寫入規約、復原範圍。Agent 必須遵循；規則衝突時以此檔為準。
- `personal-dashboard-DESIGN.md`: 決策紀錄。選型理由、被否決的方案、已驗證與未驗證的事實。

All three are maintained on GitHub `beta`; no Notion documentation mirror is maintained.