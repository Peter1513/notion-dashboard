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

目前 workspace 只有 `Log` 一張表，位於頂層。完整結構尚未建立。

目標結構：

```text
Peter Wang's Space
└─ Dashboard                  page
   ├─ Goals                   database
   ├─ Tasks                   database
   └─ Log                     database
```

Schema 欄位定義見 `personal-dashboard-OPERATIONS.md` §2。建表時以該檔為準，不要憑記憶填欄位。

GUI-operation-pseudocode:

```text
1. Open Notion
2. In sidebar, click + next to Private
3. Name the page "Dashboard"
4. Inside Dashboard, type /database and select "Table - Full page"
5. Name it "Goals"; add properties per OPERATIONS §2
6. Repeat for "Tasks"
7. Drag existing "Log" database into Dashboard
8. Confirm all three appear nested under Dashboard in sidebar
```

亦可請已連線的 agent 執行；agent 須先讀 OPERATIONS。

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

### 3. 建立排程路徑 token

僅在需要無人值守寫入時執行。hosted MCP 不支援非互動授權，故排程必須走 REST API + token。

GUI-operation-pseudocode:

```text
1. Open the Notion developer portal
2. Create a new internal connection, bound to this workspace
3. In Capabilities, enable ONLY "Read content" and "Insert content"
4. Leave "Update content" disabled
5. Copy the installation access token from the Configuration tab
6. Open the Dashboard page in Notion
7. Click ••• > Add connections > select the new connection
8. Confirm child databases inherit the connection
```

存入環境變數：

```bash
export NOTION_TOKEN='<installation-access-token>'
```

Expected result:

```text
變數已設定；token 不進原始碼、不進 git
```

未 connect 的頁面，API 請求一律回錯。若 token 可讀到 Dashboard 以外的內容，代表 connect 範圍錯了，需回到步驟 6 檢查。

### Deploy Notes

```text
Free 方案單檔上傳上限 5 MB。Evidence 欄只放 URL，不上傳檔案。
不勾 Update content 是本專案唯一的硬約束；勾了就失去防護。
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

```bash
curl -fsS -X POST https://api.notion.com/v1/search \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"page_size":1}'
```

Expected result:

```text
HTTP 200 與 JSON results 陣列；未 connect 時回 object_not_found
```

此指令尚未在本專案實測，`Notion-Version` 值須依你建立 connection 時的 API 版本調整。

### Restart

連線失效時的處理是重新授權，非重啟：

```bash
claude mcp remove notion
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

Expected result:

```text
移除後重加；於 Claude Code 內 /mcp 重走 OAuth
```

## Operation Guide

### Common Workflow

```text
1. 開啟任一 agent，確認 Notion 已連線
2. 讓 agent 讀 OPERATIONS，取得 schema 與規約
3. 讀 view: Agent: Log 14d / Agent: Open Tasks / Agent: Active Goals
4. 寫入 Log 或 Tasks，每列帶 OpID 與 Source
5. 每週匯出 CSV 進 git repo
```

### Common Commands

Agent 讀取一律走 view mode。以 Claude Code 為例，指示 agent：

```text
Read the "Agent: Log 14d" view via query_data_sources with mode "view".
Do not run SQL.
```

Expected result:

```text
回傳該 view 的列，不消耗 SQL 配額
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
1. Edit personal-dashboard-OPERATIONS.md, bump its version
2. Commit to git
3. Apply the change in Notion
4. Re-run the smoke test in Verification
5. If Notion and OPERATIONS disagree, OPERATIONS wins; fix Notion
```

反向操作（先改 Notion 再補文件）會讓 agent 依過期規約寫入，是本專案最常見的失效模式。

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
Call notion-query-data-sources with mode "view" on the "Agent: Log 14d" view.
```

Expected result:

```text
回傳該 view 的列陣列；空表時回傳空陣列而非錯誤
```

### Manual Verification

```text
1. Open Notion on mobile
2. Navigate to the Log database
3. Confirm the row just written by the agent appears
4. Confirm its Source and OpID match what the agent reported
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

以下為 2026-09-02 實際執行結果，非預期值。

```text
Command:
notion-fetch id="self"

Status:
ok

Output summary:
workspace "Peter Wang's Space" (754f2ec5-7392-818d-8cfa-000384801235)
query_data_sources: available_with_limit
query_multiple_data_sources: full_version_required
query_meeting_notes: plan_required
all other tools: available

Result:
PASS
```

```text
Command:
notion-query-data-sources mode="view" view_url="view://3cff2ec5-7392-8164-bfa0-000cec75fadc"

Status:
ok

Output summary:
1 row returned; Source=codex, OpID=p2-20260902-1041-chatgpt-001

Result:
PASS
```

```text
Command:
notion-create-pages (draft, DESIGN page)

Status:
ok

Output summary:
page created, id 3cff2ec5-7392-8120-a3da-fdfcf1e3a2e4

Result:
PASS
```

```text
Command:
notion-create-pages (draft, OPERATIONS page)

Status:
error

Output summary:
No approval received

Result:
FAIL
```

尚未執行、僅為預期值的項目：排程路徑的 `curl` 檢查、`Agent: Open Tasks`、`Agent: Active Goals`、`Agent: OpID Lookup` 三個 view（尚未建立）。

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

### create_pages returns "No approval received"

Symptom:

```text
No approval received
```

Diagnosis:

```text
Compare against a prior successful create_pages call in the same session.
If parameters are equivalent, the failure is client-side approval, not the payload.
```

Likely cause:

```text
客戶端的寫入核准機制未通過。已觀察到同 session 內先成功後失敗。
```

Fix:

```text
1. Retry the call
2. If it fails again, create the page manually in Notion
3. Or import the markdown file: Notion > Import > Markdown
```

Verify:

```text
Call notion-fetch on the new page ID and confirm content.
```

Expected result:

```text
回傳頁面內容
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

### Log view returns nothing or stale rows

Symptom:

```text
Agent: Log 14d 回傳空陣列或明顯過舊的列
```

Diagnosis:

```text
Fetch the Log database and read the view's advancedFilter value.
```

Likely cause:

```text
該 view 目前使用 hardcoded 日期（2026-08-19），非滾動 14 天視窗。時間一久即漂移。
```

Fix:

```text
1. Open the Log database in Notion
2. Open the "Agent: Log 14d" view filter
3. Change the Date condition to a relative range covering the past 14 days
```

Verify:

```text
Call notion-query-data-sources with mode "view" on the view again.
```

Expected result:

```text
回傳近 14 天的列
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
