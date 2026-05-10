# Ntropii MCP tool reference (for Claude Managed Agents)

The MCP tool surface a Claude MA invokes to drive a Ntropii task. Auth is handled by Anthropic's runtime via the Vault attached at session start — agents do not pass API keys explicitly.

This page is the agent-author-facing surface only. Worker-side wiring (the Anthropic adapter, sessions table, api-tenant `/events` ingress) is documented in the Ntropii internal repos.

## Status

Phase 2 / Linear N-76. Existing ntro-mcp tools (`ntro_task_*`, `ntro_runbook_*`, etc.) are live; the Phase 2 agent-runtime tools (`ntro_steps_*`, `ntro_tasks_get_period`, `ntro_tasks_list_events`) are scoped under N-76 section C.

---

## Steps tools — `ntro_steps_*`

Drive the Ntropii breadcrumb. Steps the agent declares appear nested under the parent runbook step.

### `ntro_steps_define(steps: list[dict]) -> None`

Declare the agent's plan. Call once at the start of the Run.

```json
{
  "steps": [
    {"id": "read_period",    "title": "Read period data",         "icon": "BookOpen"},
    {"id": "draft_markdown", "title": "Draft handover narrative", "icon": "FileText"},
    {"id": "render_docx",    "title": "Render handover .docx",    "icon": "Download"}
  ]
}
```

Each step entry: `{id, title, icon?}`. Icons are [Lucide](https://lucide.dev) names; the UI falls back to a default if unknown.

### `ntro_steps_progress(step_id, status, payload?, display_hint?) -> None`

Update a step's status. Statuses: `active`, `completed`, `failed`.

```json
{
  "step_id": "read_period",
  "status": "completed",
  "payload": {"period": "<period>", "entity": "<entity-slug>"}
}
```

`payload` is free-form JSON surfaced in the breadcrumb drill-in. `display_hint` (optional) controls how the payload renders — e.g. `{"component": "DATA_TABLE", "config": {...}}` to render a table.

### `ntro_steps_request_file(source, schema, ...) -> FileRef`

Open a HITL gate that asks the human user to upload a file matching the given source/schema labels. The call blocks until the upload is ingested.

```json
{"source": "zenko-rent-roll", "schema": "rent-roll-zenko"}
```

### `ntro_steps_request_approval(payload, display_hint?) -> dict`

Open a HITL gate that asks the user to approve or reject. The call blocks until the user responds.

```json
{
  "payload": {"draft_url": "...", "summary": "..."},
  "display_hint": {"component": "DATA_TABLE", "config": {}}
}
```

Returns:
```json
{"action": "approve", "reason": null, "actedBy": "user@example.com"}
```

---

## Tasks tools — `ntro_tasks_*`

Read the current task's context.

### `ntro_tasks_get_period() -> dict`

Returns the period summary the parent runbook has accumulated so far.

```json
{
  "entity": {"id": "...", "slug": "<entity-slug>", "name": "...", "currency": "GBP"},
  "period": "2026-03",
  "tb": {"opening_total": "...", "closing_total": "...", "lines": []},
  "journal_proposal": {"lines": [], "totals": {}, "checks": {}},
  "documents": [{"source": "...", "filename": "...", "document_ref": "..."}],
  "checks": {"journal_balanced": {"status": "pass", "detail": "..."}}
}
```

### `ntro_tasks_list_events(since?: str) -> list[dict]`

Returns the parent task's StepEvent stream — useful for narrating what happened during the run.

```json
[{"type": "step_completed", "step_id": "...", "at": "...", "payload": {}}]
```

### `ntro_tasks_list_files(period?: str, source_prefix?: str) -> list[dict]`

Enumerate files attached to the current task in the tenant data plane.

```json
[{"id": "...", "filename": "...", "source": "zenko-rent-roll", "uploaded_at": "..."}]
```

---

## Existing tools the agent can use

All current ntro-mcp tools work too — see the live ntro-mcp tool list. Useful ones for agents:

- `ntro_runbook_get(slug)` — read a runbook's config schema (rare; the period summary usually has what you need)
- `ntro_task_get(task_id?)` — same task summary, lower-level
- `ntro_task_next_step(task_id?)` — generic action loop (mostly used by Claude Code on the human side)

If `task_id` is omitted, MCP tools resolve to the current session's task automatically — agents don't need to track ids.

---

## What is NOT exposed

These deliberately don't exist. If you need them, document the use case and the surface can be extended.

- **`ntro_files_persist(bytes, ...)` (agent push):** file passback is adapter-driven (attach file to Run output, Ntropii pulls it at completion). Agents don't push.
- **Cross-task / cross-tenant queries:** an agent sees only the task it was invoked for.
- **GL writes / journal commits:** posting journals is the runbook's responsibility, not the agent's.

---

## Worked example

See [`agents/audit-handover/system-prompt.md`](../agents/audit-handover/system-prompt.md) for a complete agent that uses `ntro_steps_*` + `ntro_tasks_get_period` + `ntro_tasks_list_events` to produce a .docx output.
