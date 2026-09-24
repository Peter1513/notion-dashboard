### 共用

```python
def resolve(database, ref, for_write=False):
    if is_record_id(ref):
        record = fetch_record(ref)
    else:
        record = require_unique_match(
            find_records(database, ref if is_mapping(ref) else {"Name": ref})
        )
        if for_write:
            confirm_with_user(record)
    return record
```

### Goal

```python
def create_goal(name, status="paused"):
    require_one_of(status, ["paused", "active", "dropped", "done"])
    return create_record(GOALS, {"Name": name, "Status": status})

def update_goal(ref, name=None, status=None):
    goal = resolve(GOALS, ref, for_write=True)
    changes = {}
    if name is not None:
        changes["Name"] = name
    if status is not None:
        require_one_of(status, ["paused", "active", "dropped", "done"])
        changes["Status"] = status
    return update_record(goal, changes)

def drop_goal(ref):
    return update_goal(ref, status="dropped")
```

### Task

```python
def create_task(name, status="todo", seq=None, goal_ref=None):
    require_one_of(status, ["todo", "doing", "blocked", "dropped", "done"])
    properties = {"Name": name, "Status": status}
    if seq is not None:
        properties["Seq"] = seq
    if goal_ref is not None:
        properties["Goal"] = resolve(GOALS, goal_ref, for_write=True)
    return create_record(TASKS, properties)

def update_task(ref, name=None, status=None, seq=None, goal_ref=None):
    task = resolve(TASKS, ref, for_write=True)
    changes = {}
    if name is not None:
        changes["Name"] = name
    if status is not None:
        require_one_of(status, ["todo", "doing", "blocked", "dropped", "done"])
        changes["Status"] = status
    if seq is not None:
        changes["Seq"] = seq
    if goal_ref is not None:
        changes["Goal"] = resolve(GOALS, goal_ref, for_write=True)
    return update_record(task, changes)

def drop_task(ref):
    return update_task(ref, status="dropped")

def clear_task_goal(ref):
    task = resolve(TASKS, ref, for_write=True)
    return update_record(task, {"Goal": None})
```

### Log

```python
def create_log(name, date, delta, source, evidence=None):
    require_one_of(source, ["manual", "claude", "codex", "grok"])
    properties = {"Name": name, "Date": date, "Delta": delta, "Source": source}
    if evidence is not None:
        properties["Evidence"] = evidence
    return create_record(LOG, properties)

def update_log(ref, name=None, date=None, delta=None, evidence=None, source=None):
    record = resolve(LOG, ref, for_write=True)
    changes = {}
    if name is not None:
        changes["Name"] = name
    if date is not None:
        changes["Date"] = date
    if delta is not None:
        changes["Delta"] = delta
    if evidence is not None:
        changes["Evidence"] = evidence
    if source is not None:
        require_one_of(source, ["manual", "claude", "codex", "grok"])
        changes["Source"] = source
    return update_record(record, changes)

def clear_log_evidence(ref):
    record = resolve(LOG, ref, for_write=True)
    return update_record(record, {"Evidence": None})
```