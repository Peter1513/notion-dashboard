# README v1.2.1 — Notion Dashboard

> **來源限制**：本文件內容**僅**由 `SCHEMA_v1-2-1.yaml` 與 `OPERATIONS_v1-2-1.md` 推導。未引用 v1-1 / v2 / v1 discovery 文件，亦未查詢線上 Notion 實況。兩份來源未涵蓋者一律列於「未知 / 待查」，不臆測。

---

## 1. 系統概觀

一個 Notion page（`Dashboard`）底下掛三個 database，供多個 agent（claude / codex / grok）與人工共同讀寫。

```
Dashboard (page)
├── Goals   (database)
├── Tasks   (database)  ──relation(Goal)──> Goals
└── Log     (database)
```

模組層次：

```
Agent / 使用者
      │  呼叫 operations
      ▼
Operations layer  (create_* / update_* / drop_* / clear_*)
      │  全部先經 resolve()
      ▼
resolve(database, ref, for_write)
      │  ref 為 record id → 直取
      │  否則 find_records → require_unique_match
      │  for_write=True → confirm_with_user
      ▼
Notion records  (runtime IDs 由 binding 階段綁定)
```

關係僅一條：`Tasks.Goal → Goals`。`Log` 與 Goals / Tasks **無** relation。

---

## 2. Schema 契約：logical vs runtime

Schema 分兩層，這是 v1.2.1 的核心設計：

| 層 | 內容 | 是否為 schema identity |
|---|---|---|
| `logical` | key、title、properties、types、status/select 語意、relation target、views 定義與排序 | 是（穩定） |
| `runtime` | `page_id`、`database_id`、`data_source_id`、`view_id`、collection URI、view URI、Notion page URL | 否（instance-specific） |

`resolution.rules` 明定：

1. runtime ID 是 instance-specific，**不得**當作 schema identity。
2. logical key 才是 agent 使用的穩定識別子。
3. relation target 必須指向 logical key（本版 `tasks.goal.target = databases.goals`）。
4. 綁定 runtime ID 前，候選物件必須先對照 logical schema 驗證。

---

## 3. 資料模型

### 3.1 Goals（`databases.goals`）

| logical key | Notion 名稱 | 型別 | 備註 |
|---|---|---|---|
| name | Name | title | |
| status | Status | status | 見 §4 |
| id | ID | auto_increment_id | |

### 3.2 Tasks（`databases.tasks`）

| logical key | Notion 名稱 | 型別 | 備註 |
|---|---|---|---|
| name | Name | title | |
| status | Status | status | 見 §4 |
| seq | Seq | number | `agent_all` view 的排序鍵 |
| goal | Goal | relation | target = `databases.goals` |
| id | ID | auto_increment_id | |

### 3.3 Log（`databases.log`）

| logical key | Notion 名稱 | 型別 | 備註 |
|---|---|---|---|
| name | Name | title | |
| id | ID | auto_increment_id | |
| date | Date | date | `agent_all` view 的排序鍵 |
| delta | Delta | text | |
| evidence | Evidence | url | 可清除 |
| source | Source | select | 見 §4 |

---

## 4. 狀態與列舉語意

### Goals.Status（status，含 group）

| 值 | group |
|---|---|
| paused | to_do |
| active | in_progress |
| dropped | complete |
| done | complete |

### Tasks.Status（status，含 group）

| 值 | group |
|---|---|
| todo | to_do |
| doing | in_progress |
| blocked | in_progress |
| dropped | complete |
| done | complete |

### Log.Source（select，無 group）

`manual` / `claude` / `codex` / `grok`

> `blocked` 被歸在 `in_progress`、`dropped` 被歸在 `complete`：這是刻意的語意選擇，group 只反映「是否還在流程中」，不反映成敗。

---

## 5. Views

每個 database 兩個 view，皆為 table、filter 皆為 `none`：

| database | view key | Notion 名稱 | 欄位順序 | 排序 |
|---|---|---|---|---|
| goals | default | Default view | name, status, id | none |
| goals | agent_all | Agent: All Goals | name, status, id | none |
| tasks | default | Default view | name, status, seq, goal, id | none |
| tasks | agent_all | Agent: All Tasks | name, status, seq, goal, id | seq ascending |
| log | default | Default view | name, date, delta, evidence, id, source | none |
| log | agent_all | Agent: All Log | date, name, source, delta, evidence, id | date descending |

`agent_all` 系列是給 agent 用的「全量、無過濾」入口；排序是 view 的穩定屬性（列在 `binding_policy.stable`），不可任意更動。

---

## 6. 部署（Deployment）

依 `resolution.lookup` 與 `logical` 定義重建一個實例：

1. 建立 page，title = `Dashboard`。
2. 在該 page 底下建立三個 database：`Goals`、`Tasks`、`Log`。
3. 依 §3 建立各 database 的 properties（Notion 名稱與型別須完全吻合）。
4. 建立 `Tasks.Goal` relation，指向 `Goals`。
5. 依 §4 設定 status 選項與 group、select 選項。
6. 依 §5 建立兩組 view，含欄位順序與排序規則。
7. 回填 `runtime` 區塊：`page_id`、各 database 的 `database_id` / `data_source_id`、各 view 的 `view_id`。

> `runtime` 區塊屬於單一 workspace 實例。換 workspace / 重建 page 後必須重跑 §7 重新綁定；**不要**把舊 runtime ID 當 schema 差異看待。

---

## 7. 啟動流程（Boot / Resolution）

Agent 啟動時的綁定順序：

1. **定位 dashboard**：`type: page`、`title: Dashboard`。
2. **定位 databases**：各自以 `parent: dashboard` + `type: database` + `title`（`Goals` / `Tasks` / `Log`）查找。
3. **驗證候選**：比對 logical schema — property 語意鍵、型別、status/select 語意、relation target、view 語意鍵與欄位定義、view 排序規則。
4. **綁定 runtime ID**：驗證通過後才寫入 `page_id` / `database_id` / `data_source_id` / `view_id`。
5. 之後所有操作一律以 logical key 尋址。

驗證失敗時的行為（中止 / 重建 / 告警）**來源未定義** → 見 §10。

---

## 8. 操作介面（Operations）

### 8.1 共用：`resolve(database, ref, for_write=False)`

```python
if is_record_id(ref):          # 直接以 record id 取得
    record = fetch_record(ref)
else:                          # mapping 則整組當查詢條件，否則視為 {"Name": ref}
    record = require_unique_match(find_records(database, ...))
    if for_write:
        confirm_with_user(record)
return record
```

兩個關鍵不變量：

- **非唯一即失敗**：以名稱尋址時，`require_unique_match` 要求恰好一筆命中。
- **寫入前需人工確認**：只有「以名稱／條件解析 + for_write=True」才觸發 `confirm_with_user`；直接給 record id 的路徑**不**觸發確認。

### 8.2 Goals

| 函式 | 參數 | 驗證 | 行為 |
|---|---|---|---|
| `create_goal` | name, status=`"paused"` | status 白名單 | 建立記錄 |
| `update_goal` | ref, name?, status? | 同上（有給才驗） | 僅更新有給的欄位 |
| `drop_goal` | ref | — | 等同 `update_goal(ref, status="dropped")` |

### 8.3 Tasks

| 函式 | 參數 | 驗證 | 行為 |
|---|---|---|---|
| `create_task` | name, status=`"todo"`, seq?, goal_ref? | status 白名單 | `goal_ref` 以 `for_write=True` 解析 |
| `update_task` | ref, name?, status?, seq?, goal_ref? | 同上 | 僅更新有給的欄位 |
| `drop_task` | ref | — | 等同 `update_task(ref, status="dropped")` |
| `clear_task_goal` | ref | — | 將 `Goal` 設為 `None` |

### 8.4 Log

| 函式 | 參數 | 驗證 | 行為 |
|---|---|---|---|
| `create_log` | name, date, delta, source, evidence? | source 白名單 | name/date/delta/source 為必填 |
| `update_log` | ref, name?, date?, delta?, evidence?, source? | source 白名單（有給才驗） | 僅更新有給的欄位 |
| `clear_log_evidence` | ref | — | 將 `Evidence` 設為 `None` |

### 8.5 `None` 的兩種語意（易踩）

- 在 `update_*` 的**參數**上：`None` = 「不變更此欄位」。
- 在送往 `update_record` 的 **changes** 裡：`None` = 「清空此欄位」。

因此「清空」不能靠 `update_*`，必須走專用函式：`clear_task_goal`、`clear_log_evidence`。`Goals` 無任何可清空欄位，故無對應函式。

---

## 9. 不變量摘要

- 所有 status / source 寫入皆經 `require_one_of` 白名單，無自由字串。
- 預設值：goal = `paused`；task = `todo`。
- `drop_*` 是軟刪除（狀態轉為 `dropped`），來源中**沒有**硬刪除 / archive 操作。
- Tasks 可不掛 Goal（`goal_ref` 為選填，且可事後清除）。
- Log 為獨立事件記錄，與 Goals / Tasks 無結構關聯；既有 Log 可更新，`Evidence` 可透過 `clear_log_evidence` 清除。
- 跨實例可攜的只有 logical 層；runtime 層一律重綁。

---

## 10. 已知 / 未知 / 假設 / 待查

**已知**（兩份來源明確定義）：schema 版本、logical/runtime 分層、三 database 的屬性與型別、status/select 白名單與 group、view 欄位與排序、resolution lookup 規則、binding policy、全部 operations 簽章與驗證。

**未知**（來源未涵蓋，本文件不臆測）：

- Notion API 綁定實作 —— `create_record` / `update_record` / `find_records` / `fetch_record` / `is_record_id` / `is_mapping` / `require_unique_match` / `require_one_of` / `confirm_with_user` 皆只有呼叫端，無實作。
- 認證與權限（integration token、page 分享範圍）。
- 錯誤處理、重試、分頁、速率限制、並行寫入衝突。
- `find_records` 的匹配語意（精確相等 vs 模糊比對、大小寫、空白處理）。
- `require_unique_match` 命中 0 筆或多筆時的實際行為。
- `confirm_with_user` 的介面形式與拒絕後的流程。
- §7 步驟 3 驗證失敗時的處置。
- `Log.Date` 的格式與時區。
- `Tasks.Seq` 的配號規則、是否唯一、是否需重排。
- Goals 與 Tasks 是否有查詢 / 列表 / 讀取類 operation（來源只有寫入類）。
- 遷移路徑（自任何舊版升級至 1.2.1）。

**假設**（為讀懂來源所作，未經驗證）：`GOALS` / `TASKS` / `LOG` 三個常數對應 schema 的 `databases.goals` / `databases.tasks` / `databases.log`。

**待查**：本文件僅依兩份來源撰寫，尚未與線上 `Dashboard` 實況核對；runtime ID 是否仍有效未驗證。 2026-09-24 Dashboard_beta renamed to Dashboard; old Dashboard (3cff2ec5-7392-81c8-9f47-c7bebfb143eb) is in trash.