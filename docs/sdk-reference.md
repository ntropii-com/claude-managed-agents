# SDK reference — `ntro.workflow.*` for Claude Managed Agents

The runtime API a Claude MA imports from the published `ntro` package (PyPI, ≥ 0.2.0) to drive a Ntropii task. All methods are async; auth + session resolution are implicit via env vars (`NTRO_API_KEY`, `NTRO_TENANT_URL`, `NTRO_SESSION_ID`).

This page is the agent-author-facing surface only. Worker-side wiring (the Anthropic adapter, sessions table, api-tenant `/events` ingress) is documented in the Ntropii internal repos.

## Status

Phase 2 / [N-76](https://linear.app/byng/issue/N-76). Not yet in the published `ntro` 0.1.0 — agents that use these methods cannot complete a Run end-to-end until N-76 ships and `ntro` 0.2.0 is on PyPI.

---

## `ntro.workflow.steps`

Drive the Ntropii breadcrumb. Steps the agent declares appear nested under the parent runbook step.

### `define(steps: list[dict]) -> None`

Declare the agent's plan. Call once at the start of the Run.

```python
await ntro.workflow.steps.define([
    {"id": "read_period",    "title": "Read period data",         "icon": "BookOpen"},
    {"id": "draft_markdown", "title": "Draft handover narrative", "icon": "FileText"},
    {"id": "render_docx",    "title": "Render handover .docx",    "icon": "Download"},
])
```

Each step entry: `{id, title, icon?}`. Icons are [Lucide](https://lucide.dev) names; the UI falls back to a default if unknown.

### `progress(step_id: str, status: str, payload: dict | None = None, display_hint: dict | None = None) -> None`

Update a step's status. Statuses: `active`, `completed`, `failed`.

```python
await ntro.workflow.steps.progress(
    "read_period", "completed",
    payload={"period": "2026-03", "entity": "4-high-court-limited"},
)
```

`payload` is free-form JSON surfaced in the breadcrumb drill-in. `display_hint` (optional) controls how the payload renders — e.g. `{"component": "DATA_TABLE", "config": {...}}` to render a table.

### `request_file(*, source: str, schema: str, **kwargs) -> FileRef`

Open a HITL gate that asks the human user to upload a file matching the given source/schema labels. The call awaits the upload and returns when the file is ingested.

```python
ref = await ntro.workflow.steps.request_file(
    source="zenko-rent-roll",
    schema="rent-roll-zenko",
)
```

### `request_approval(payload: dict, display_hint: dict | None = None) -> dict`

Open a HITL gate that asks the user to approve or reject. The call awaits the response and returns the user's decision.

```python
decision = await ntro.workflow.steps.request_approval(
    payload={"draft_url": "...", "summary": "..."},
    display_hint={"component": "DATA_TABLE", "config": {...}},
)
# decision = {"action": "approve" | "reject", "reason": str | None, "actedBy": str}
```

---

## `ntro.workflow.tasks`

Read the current task's context.

### `get_period() -> dict`

Returns the period summary the parent runbook has accumulated so far.

```python
period = await ntro.workflow.tasks.get_period()
# {
#   "entity": {"id": "...", "slug": "4-high-court-limited", "name": "...", "currency": "GBP"},
#   "period": "2026-03",
#   "tb": {"opening_total": ..., "closing_total": ..., "lines": [...]},
#   "journal_proposal": {"lines": [...], "totals": {...}, "checks": {...}},
#   "documents": [{"source": "...", "filename": "...", "document_ref": "..."}, ...],
#   "checks": {"journal_balanced": {"status": "pass", "detail": "..."}, ...},
# }
```

### `list_events(*, since: str | None = None) -> list[dict]`

Returns the parent task's StepEvent stream — useful for narrating what happened during the run.

```python
events = await ntro.workflow.tasks.list_events()
# [{"type": "step_completed", "step_id": "...", "at": "...", "payload": {...}}, ...]
```

### `list_files(*, source_prefix: str | None = None) -> list[dict]`

Enumerate files attached to the current task in the tenant data plane.

```python
files = await ntro.workflow.tasks.list_files(source_prefix="zenko-")
# [{"id": "...", "filename": "...", "source": "zenko-rent-roll", "uploaded_at": "..."}, ...]
```

---

## What is NOT in the SDK

These deliberately don't exist. If you need them, document the use case and the SDK can be extended.

- **`files.persist(bytes, ...)` (agent push):** file passback is adapter-driven (attach file to Run output, Ntropii pulls it at completion). Agents don't push.
- **`files.read(ref) -> bytes` (agent pull):** the period summary already includes the documents the agent needs; if you need to dig into a specific PDF's bytes, propose adding `tasks.read_document(ref)`.
- **Cross-task / cross-tenant queries:** an agent sees only the task it was invoked for.
- **GL writes / journal commits:** posting journals is the runbook's responsibility, not the agent's.

---

## Worked example

See [`agents/audit-handover/system-prompt.md`](../agents/audit-handover/system-prompt.md) for a complete agent that uses `define` + `progress` + `tasks.get_period` + `tasks.list_events` to produce a .docx output.
