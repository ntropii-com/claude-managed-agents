# audit-handover Claude MA — provisioning + registration

Produces a Ntropii-style auditor handover .docx for a completed NAV close.

## Setup checklist

These are one-time steps to get the agent live on Anthropic + registered with Ntropii. Steps 1–3 are on the Anthropic side; step 4 is the Ntropii registration; step 5 wires it into a runbook.

### 1. Vault — store the Ntropii API key

Anthropic Console → Vaults → Create. The vault holds the Ntropii API key (filled in step 4 below).

### 2. Environment — pip packages + networking

Anthropic Console → Environments → Create. In the **Packages** section, set the manager to `pip` and add:

```
ntro python-docx
```

Pin if you want: `ntro==0.1.0 python-docx==1.1.0`. Networking should allow outbound to your Ntropii MCP server URL.

### 3. Agent — paste from this folder

Anthropic Console → Agents → Create. Map fields directly from this folder's `agent.yaml`:

| Console field | Source |
|---|---|
| **Name** | `audit-handover` |
| **Description** | from `agent.yaml.description` |
| **Model** | `claude-sonnet-4-6` |
| **System message** | paste the full contents of `system-prompt.md` |
| **Tools** | `agent_toolset_20260401` with `bash`, `write`, `read` enabled, plus an `mcp_toolset` entry referencing the `ntropii` MCP server |
| **MCP servers** | one entry: `{type: url, name: ntropii, url: <Ntropii MCP URL>}` |
| **Skills** | Anthropic pre-built `docx` (optional but recommended for richer document generation) |

After creation, capture the resulting `agent_id` (e.g. `agent_01HX…`).

### 4. Register the agent with Ntropii

```bash
ntro agent create \
  --path "anthropic://agents/<agent_id>" \
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

Take the `apiKey` and store it in the **Vault** from step 1. Anthropic's runtime will inject it into MCP server auth at session start.

### 5. Wire into nav-monthly-pack

Add the agent id to byng's 4HC entity config so the runbook step picks it up:

```bash
# In the workspace API:
# PATCH /workspace/tenants/<byng>/entities/<4hc> with config.audit_agent_id=<id>
```

The runbook's `draft_auditor_handover` step reads `ctx.config.audit_agent_id` and calls `ntro.workflow.agents.invoke(...)`. Ntropii's worker-side Anthropic adapter starts the session with `{agent_id, environment_id, vault_ids}` so all three are connected.

## Files in this folder

| File | Purpose |
|---|---|
| `agent.yaml` | Anthropic API-shaped metadata. Mirror values into the Console form, or POST it as the body of `/v1/agents`. |
| `system-prompt.md` | The agent's system message |
| `tools.md` | Tools + MCP server + skills reference |
| `handover-template.md` | Markdown skeleton (also inlined into `system-prompt.md` since the agent has no file system access at runtime — keep them in sync) |
| `README.md` | This file |

## Updates

When you change any of these source files:
- Edit the corresponding Console field; the agent_id stays the same, just the version bumps.
- No need to re-register with Ntropii unless the agent_id itself changes.
- If you change pip packages, update the **Environment** (not the agent).

## What works today vs after N-76

| Capability | Today | After N-76 |
|---|---|---|
| Register agent shell on Anthropic | ✓ | ✓ |
| Register agent with Ntropii (`ntro agent create`) | ✗ (CLI subcommand not wired) | ✓ |
| Agent invoked from a runbook step | ✗ (no `ntro.workflow.agents.invoke`) | ✓ |
| Phase 2 MCP tools (`ntro_steps_*`, `ntro_tasks_get_period`, `ntro_tasks_list_events`) | ✗ (not yet on ntro-mcp) | ✓ |
| Run-output file passback | ✗ (no adapter) | ✓ |
| Breadcrumb child steps from agent | ✗ (no StepEvent ingress) | ✓ |
