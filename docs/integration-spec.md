# Integration spec — Claude Managed Agents ↔ Ntropii

The contract a Claude MA satisfies when it integrates with Ntropii. Stable; agent code that follows this works regardless of which Ntropii tenant invokes it.

## 1. Runtime environment

When Ntropii invokes a Claude MA, it sets these env vars in the agent's Python execution environment:

| Var | Source | Notes |
|---|---|---|
| `NTRO_API_KEY` | Returned by `ntro agent create` once at registration; you copy it into the Console's Environment tab | Workspace-scoped API key. Required. |
| `NTRO_TENANT_URL` | Tenant URL (e.g. `https://byng.test.ntropii.com/v1`); set in the Console once at registration | Required. |
| `NTRO_SESSION_ID` | Set per-Run by Ntropii's Anthropic adapter (Phase 2 / N-76) | Identifies the current invocation. The SDK reads this implicitly — your code does not pass it explicitly. |

The SDK reads these env vars at module import. Auth is automatic.

## 2. Toolset + dependencies

Claude Managed Agents expose a single unified toolset (`agent_toolset_20260401`) with sub-tools you enable selectively. There is no separate Python sandbox tool and no native docx/xlsx/pdf skill — agents do everything via `bash`-invoked Python scripts.

Recommended tools for a Ntropii agent:

| Tool | Why |
|---|---|
| `bash` | Run `python …` and any system commands |
| `write` | Persist Python scripts and intermediate artefacts (markdown drafts, JSON dumps) to the sandbox filesystem |
| `read` | Read intermediates between steps |

Do not enable `web_fetch` / `web_search` / `computer` unless your agent genuinely needs them — they add latency and risk.

Required pip dependency: `ntro>=0.2.0` (Phase 2 / N-76 release) for the `ntro.workflow.*` SDK. Add domain-specific deps as needed (e.g. `python-docx`, `openpyxl`, `pandas`).

## 3. Lifecycle

A Claude MA invocation is a single Anthropic Run started by Ntropii's adapter (worker-side). Within a Run, the agent:

1. **Declares its plan** via `ntro.workflow.steps.define([{id, title, icon}, ...])`. This populates the Ntropii breadcrumb so the human user sees what the agent is doing in real time. Each declared id can be progressed individually below the parent step in the breadcrumb.
2. **Reads task context** via `ntro.workflow.tasks.get_period()`, `list_events()`, `list_files()`. The data is whatever the Ntropii task has at this point — TB, journal proposal, ingested documents, the StepEvent stream from the parent runbook.
3. **Progresses each step** via `ntro.workflow.steps.progress(step_id, status, payload?, display_hint?)`. Status values: `active`, `completed`, `failed`. Payload + display_hint are optional and surface in the breadcrumb's drill-in.
4. **Optionally requests human input** via `ntro.workflow.steps.request_file(...)` (gates the run waiting on a file upload) or `request_approval(...)` (gates waiting on approve/reject). The SDK call returns when the human responds.
5. **Attaches output files** to the Anthropic Run output. Ntropii's adapter automatically pulls them at Run completion and persists to the tenant data plane (`ingest.submitted_documents` with `source='agent_output:<agent_id>'`). The agent does NOT call any Ntropii persist/upload function.
6. **Returns a one-line summary** as its final assistant message. Ntropii surfaces it in the parent runbook step's HITL review screen.

## 4. File passback

The agent produces files (.docx, .xlsx, .pdf, .csv, etc.) by:

1. Generating the file in its sandbox (e.g. `python-docx` writes to `/tmp/foo.docx`).
2. Attaching the file to its Run output (Anthropic's standard mechanism — the file shows up in the Run's `output.files`).

Ntropii's worker-side Anthropic adapter:

1. Polls the Run for completion.
2. On completion, fetches each attached file via Anthropic's Files API.
3. Persists each via `ntro.ingest.insert_submitted_document(...)` to the tenant data plane.
4. Returns the persisted FileRefs to the calling runbook step as `AgentResult.output_files`.

The agent does **not** call `ntro.workflow.files.persist(...)` — that helper does not exist in Phase 2. File passback is exclusively adapter-driven.

## 5. Error semantics

- **Deterministic-input failures** (pydantic ValidationError, ValueError on bad inputs, etc.) raised inside the SDK reach the worker layer; the worker converts them into non-retryable `ApplicationError`s so the Run fails fast rather than retrying forever. The agent does not need to wrap these.
- **Transient errors** (network, rate limit) are retried per the activity's retry policy.
- **Agent-side exceptions** in the Python sandbox should be surfaced in the agent's final assistant message AND should NOT mark a step `completed` it didn't actually finish. Use `progress(step_id, "failed", payload={"error": "..."})` to indicate failure cleanly.

## 6. What the agent does NOT do

- **Tenant or entity routing.** The current invocation is implicit via `NTRO_SESSION_ID` — agents do not select tenants or entities.
- **Direct DB access.** All data access is via `ntro.workflow.tasks.*` HTTP calls. There is no DB connection from the agent's sandbox.
- **Cross-task visibility.** An agent sees only the period+task it was invoked for, not other tenants/entities/historical runs (unless the SDK surface is extended).
- **Schema authoring.** Agents fill in deterministic templates; they do not invent data shapes.
