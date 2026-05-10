# audit-handover Claude MA — provisioning + registration

Source artefacts for byng's `audit-handover` Claude Managed Agent. Three steps to get it running end-to-end:

1. **Provision** the agent on Anthropic via the Console (returns an `agent_id`).
2. **Register** the deployed agent with Ntropii (returns an Ntropii API key).
3. **Plug** the Ntropii API key back into the agent's env config so it can call the SDK.

After all three, the byng `nav-monthly-pack` runbook can invoke the agent via `ntro.workflow.agents.invoke(agent_id, ...)`.

---

## 1. Provision on Anthropic (Console)

Anthropic does not yet ship a dedicated CLI for Managed Agents — provisioning is via the Console at <https://console.anthropic.com> (or via the `/v1/agents` API directly with the `anthropic-beta: managed-agents-2026-04-01` header, but the Console is more ergonomic for one-offs).

### Console fields

Open the Console → Agents → Create. Map fields from the source artefacts in this folder. `agent.yaml` uses the same field names as Anthropic's API (`name`, `model`, `system`, `tools`, `execution_environment`) so you can copy values directly into the matching Console inputs:

| Console field | Source |
|---|---|
| **Name** | `audit-handover` (`agent.yaml.name`) |
| **Description** | from `agent.yaml.description` |
| **Model** | `claude-sonnet-4-6` (`agent.yaml.model`) |
| **System message** | paste the full contents of `system-prompt.md` |
| **Toolset** | `agent_toolset_20260401` — enable only `bash`, `write`, `read` (see `tools.md`) |
| **Pip dependencies** | from `agent.yaml.execution_environment.python.pip` — `ntro>=0.1.0,<0.2.0` and `python-docx>=1.1.0` |
| **Environment variables** | `NTRO_API_KEY`, `NTRO_TENANT_URL` (placeholder values for now), `NTRO_SESSION_ID` (left blank — Ntropii sets it per-Run) |

After creation, capture the `agent_id` (e.g. `agent_01HX…`).

> **Heads-up on `ntro` SDK readiness.** `ntro` is on PyPI (`0.1.0`) and pip-install will succeed. However the Managed Agent's process loop calls `ntro.workflow.steps.*` and `ntro.workflow.tasks.*` — those modules are Phase 2 / N-76 work and NOT in the published `0.1.0`. The agent will register cleanly today, but cannot actually drive a Run end-to-end until N-76 ships and a new `ntro` version is published. Bump `agent.yaml.execution_environment.python.pip` (and the Console field) to that version when it lands.

> **Why Console (or API direct), not a YAML import.** Anthropic's Console enforces its own field schema and rejects non-matching keys. `agent.yaml` is shaped to match that schema so it's useful as paste-from reference; if you later want to script provisioning, POST it as the body of `/v1/agents` (with the `anthropic-beta: managed-agents-2026-04-01` header). API path documented at <https://platform.claude.com/docs/en/managed-agents/agent-setup.md>.

---

## 2. Register with Ntropii

```bash
ntro agent create \
  --path "anthropic://agents/<agent_id_from_step_1>" \
  --tenant byng \
  --name audit-handover
```

This returns:

```json
{
  "id": "...",
  "kind": "claude_managed",
  "externalRef": "anthropic://agents/<agent_id>",
  "apiKey": "ntro_api_key_…"   ← shown ONCE; not retrievable later
}
```

Save the `apiKey`.

---

## 3. Plug the Ntropii API key + tenant URL into the agent's env

Back in the Console → your agent → **Environment variables**, set:

| Var | Value |
|---|---|
| `NTRO_API_KEY` | the `apiKey` from step 2 |
| `NTRO_TENANT_URL` | `https://byng.test.ntropii.com/v1` |
| `NTRO_SESSION_ID` | leave blank — Ntropii adapter sets it per Run |

Save. The agent now has everything it needs to call back into Ntropii.

---

## 4. Wire into nav-monthly-pack

Add the agent id to byng's 4HC entity config so the runbook step picks it up:

```bash
# In the workspace API:
# PATCH /workspace/tenants/<byng>/entities/<4hc> with config.audit_agent_id=<id>
```

The runbook's `draft_auditor_handover` step reads `ctx.config.audit_agent_id` and calls:

```python
result = await ntro.workflow.agents.invoke(
    ctx.config.audit_agent_id,
    input={"period": ctx.period, "entity_id": ctx.entity_id},
)
```

The Anthropic adapter handles the Run lifecycle, file passback, and StepEvent fan-out automatically. (Phase 2 / N-76 implements the SDK + adapter — these source artefacts and the registered agent are deployable in advance.)

---

## Updates

When you change any of these source files (`system-prompt.md`, `tools.md`, `agent.yaml`, `handover-template.md`):

- Edit in the Console — the agent_id stays the same, just the version bumps.
- No need to re-register with Ntropii unless the agent_id itself changes.
- Bump `agent.yaml.pip` (`ntro` pin) whenever ntro-python ships a release the agent should pick up; remember to mirror that change in the Console's Pip dependencies field.

---

## Files in this folder

| File | Purpose |
|---|---|
| `agent.yaml` | Adapter-side metadata: model, toolset, pip, env vars |
| `system-prompt.md` | The agent's system message (paste into Console "System message") |
| `tools.md` | Toolset config + pip + env reference (mirrors agent.yaml in Console-friendly form) |
| `handover-template.md` | Markdown skeleton the agent fills in section-by-section |
| `README.md` | This file |
