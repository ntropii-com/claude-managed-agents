# Integration spec — Claude Managed Agents ↔ Ntropii

The contract a Claude MA satisfies when it integrates with Ntropii. Stable; agent code that follows this works regardless of which Ntropii tenant invokes it.

## 1. Anthropic-side shape

Three Anthropic objects need to exist on Anthropic's platform before a Claude MA can run against Ntropii:

| Object | Carries | Notes |
|---|---|---|
| **Environment** | `pip` packages, `npm` packages, networking, metadata | Independent of any specific Agent. Reusable across agents. |
| **Agent** | `name`, `model`, `system`, `tools`, `mcp_servers`, `skills`, `description`, `metadata` | The agent definition. References MCP servers + skills, but NOT environment / vault directly. |
| **Vault** | API keys + secrets | Holds the Ntropii API key. Referenced at session creation. |

A session is then started with `{agent_id, environment_id, vault_ids[]}`. Anthropic's runtime injects the vault's secrets into MCP server auth (and only there — they don't appear as env vars in the bash sandbox).

## 2. The two transports

Ntropii integration uses **both** an MCP server (for auth-required calls) and pip packages (for sandbox-side helpers). They serve different roles.

### MCP transport — `ntro-mcp`

The Ntropii MCP server exposes all tools that hit the workspace API:

- Existing tools: `ntro_task_get`, `ntro_task_next_step`, `ntro_task_file_ingest`, `ntro_runbook_get`, `ntro_workflow_run`, etc.
- Optional agent-runtime tools (when richer breadcrumb / mid-run HITL is needed): `ntro_steps_define`, `ntro_steps_progress`, `ntro_steps_request_file`, `ntro_steps_request_approval`, `ntro_tasks_get_period`, `ntro_tasks_list_events`, `ntro_tasks_list_files`.

Auth happens at session start: the Vault holding the Ntropii API key is referenced by the session, and Anthropic's runtime injects it into MCP server auth headers automatically. The agent doesn't see or pass the key.

In the agent's `tools`, declare:
```yaml
- type: mcp_toolset
  mcp_server_name: ntropii
```

In the agent's `mcp_servers`:
```yaml
- type: url
  name: ntropii
  url: https://mcp.ntropii.com/v1
```

### pip transport — sandbox-side `ntro` + `python-docx`

The Environment's pip packages get installed in the agent's bash Python sandbox. These are for **non-auth-requiring** local work:

- `ntro` — types (`ntro.types.wire.StepEvent`, `ntro.subledger.types.*`), local data utilities. Useful for validating MCP responses against typed Pydantic models.
- `python-docx` — render the final .docx in the sandbox. Alternative to using Anthropic's pre-built `docx` skill.

SDK methods (`ntro.workflow.steps.*`, etc.) aren't needed in the pip package — the same surface lives on the MCP server. Future ntro releases may add a thin Python wrapper over the MCP server for ergonomic typing, but it's not required.

## 3. Lifecycle

A Claude MA invocation is a single Anthropic Run started by Ntropii's adapter (worker-side). Within a Run, the agent:

1. **Declares its plan** by calling `ntro_steps_define` MCP tool with `[{id, title, icon}, ...]`. This populates the Ntropii breadcrumb so the human sees what the agent is doing in real time.
2. **Reads task context** by calling `ntro_tasks_get_period`, `ntro_tasks_list_events`, `ntro_tasks_list_files`. These return the period summary, the parent task's StepEvent stream, and the index of files attached to the task.
3. **Progresses each step** by calling `ntro_steps_progress(step_id, status, payload?, display_hint?)`. Status values: `active`, `completed`, `failed`.
4. **Optionally requests human input** via `ntro_steps_request_file(...)` (waits on file upload) or `ntro_steps_request_approval(...)` (waits on approve/reject). The MCP call returns when the human responds.
5. **Renders + attaches output files** to the Anthropic Run output. The agent uses python-docx (or Anthropic's `docx` skill) to render the file in `/tmp/`, then attaches it via Anthropic's standard Run-output mechanism. Ntropii's adapter pulls the attachment at Run completion and persists to the tenant data plane.
6. **Returns a one-line summary** as its final assistant message. Ntropii surfaces it in the parent runbook step's HITL review screen.

## 4. File passback

The agent does NOT push files to Ntropii directly. Instead:

1. Agent generates the file in its sandbox (e.g. `python-docx` writes to `/tmp/foo.docx`).
2. Agent attaches the file to its Run output (Anthropic's standard mechanism).
3. Ntropii's worker-side Anthropic adapter polls the Run; on completion it fetches each attached file via Anthropic's Files API and persists each via `ntro.ingest.insert_submitted_document(...)` (`source='agent_output:<agent_id>'`).
4. The persisted FileRefs come back to the calling runbook step as `AgentResult.output_files`, which flows to downstream HITL review steps via Temporal data.

There is no `ntro.workflow.files.persist(...)` MCP tool or SDK method. File passback is exclusively adapter-driven.

## 5. Error semantics

- **Deterministic-input failures** (pydantic ValidationError, ValueError on bad inputs, etc.) raised inside Ntropii activities reach the worker layer; the worker converts them into non-retryable `ApplicationError`s so the Run fails fast rather than retrying forever. The agent does not need to wrap these.
- **Transient errors** (network, rate limit on the MCP transport) are retried per the activity's retry policy.
- **Agent-side exceptions** in the Python sandbox should be surfaced in the agent's final assistant message AND should NOT mark a step `completed` it didn't actually finish. Use `ntro_steps_progress(step_id, "failed", payload={"error": "..."})` to indicate failure cleanly.

## 6. What the agent does NOT do

- **Tenant or entity routing.** The current invocation is implicit via the session — agents do not select tenants or entities.
- **Direct DB access.** All data access is via the MCP server. There is no DB connection from the agent's sandbox.
- **Cross-task visibility.** An agent sees only the period+task it was invoked for.
- **Schema authoring.** Agents fill in deterministic templates; they do not invent data shapes.
