# claude-managed-agents

Claude-specific reference + source artefacts for **Anthropic Managed Agents that integrate with Ntropii**.

The repo answers a single question: *"I'm building a Claude MA that needs to call into Ntropii — how does that work?"*

## TL;DR — the integration shape

Three Anthropic objects need to exist on Anthropic's platform:

| Anthropic object | Carries | Reusable? |
|---|---|---|
| **Environment** | `pip` packages (`ntro`, `python-docx`, etc.), networking, metadata | Yes — across many agents |
| **Agent** | `name`, `model`, `system`, `tools`, `mcp_servers`, `skills` | Per-agent |
| **Vault** | The Ntropii API key | Yes |

A **session** is started with `{agent_id, environment_id, vault_ids[]}`. Anthropic's runtime injects the Vault's secrets into MCP server auth — they do **not** appear as env vars in the agent's bash sandbox.

Two transports the agent uses:

| Transport | Use for | Auth |
|---|---|---|
| **MCP tools** (`mcp_servers` → `ntropii`) | Anything that hits Ntropii — `ntro_steps_*`, `ntro_tasks_*`, `ntro_task_get`, etc. | Vault-injected by Anthropic's runtime |
| **pip `ntro` + `python-docx`** (Environment package) | Sandbox-side helpers — types for validating MCP responses, document rendering | None (no API calls) |

## What's in this repo

| Path | Purpose |
|---|---|
| [`docs/integration-spec.md`](docs/integration-spec.md) | The contract: Anthropic objects, transports, lifecycle, file passback, error semantics |
| [`docs/sdk-reference.md`](docs/sdk-reference.md) | Ntropii MCP tool reference (steps, tasks) + which existing ntro-mcp tools agents use |
| [`agents/<name>/`](agents) | Source artefacts for a specific reference agent (system prompt, tools config, agent.yaml, README) |

## Setup checklist (one-time per agent)

1. **Create a Vault** on Anthropic holding the Ntropii API key. Vaults are global to your Anthropic workspace; one Vault can back many agents/sessions.
2. **Create an Environment** on Anthropic with pip packages: `ntro python-docx` (and any other helpers the agent needs). Networking should allow outbound to your Ntropii MCP server (`https://mcp.test.ntropii.com` or wherever it's hosted).
3. **Create the Agent** on Anthropic — paste `system-prompt.md`, declare `tools` (toolset + mcp_toolset), `mcp_servers` (pointing at ntropii), `skills` (optionally Anthropic's `docx`).
4. **Register the agent with Ntropii** — `ntro agent create --path anthropic://agents/<id> --tenant <slug>` returns a Ntropii API key. Stash it in the Vault from step 1.
5. **Wire it into a runbook** — add the agent_id to the entity's config so the runbook step picks it up. The runbook calls `ntro.workflow.agents.invoke(agent_id, ...)` which starts an Anthropic session (`{agent_id, environment_id, vault_ids}`) under the hood.

## Scope

- **Claude Managed Agents only.** Other ecosystems (OpenAI Assistants, Vertex AI Agent Builder, Azure AI Agent Service) integrate via the same Ntropii MCP server but use their own runtimes; their reference artefacts will live in sibling repos when they're built.
- **Single-tenant agent code.** Tenant-specific configuration (entity ids, period selectors, output filenames) is passed in at session-start time — agents themselves are tenant-neutral.

## Status

The architecture is locked: MCP for auth-required calls, pip for sandbox-side helpers, Vaults for secrets, Environments for runtime config. The Phase 2 work tracked in Linear N-76 is:

- Extending **ntro-mcp** with the agent-runtime tools (`ntro_steps_*`, `ntro_tasks_*`).
- Building the **Anthropic adapter** in ntro-worker that starts sessions, polls Run output, and persists agent-output files.
- Wiring the adapter from `ntro.workflow.agents.invoke` (the runbook-side helper).

The pip `ntro` package's role is supplemental (typed models for MCP responses, sandbox helpers). The agent's process loop is fully driven by MCP tools — the agent does not need a new pip release to function. A future `ntro` release may add a thin Python wrapper over the MCP transport for ergonomic typing.

## Reference agents

| Agent | Description | Status |
|---|---|---|
| [`agents/audit-handover/`](agents/audit-handover) | Produces an auditor handover .docx for a completed NAV close. First demo agent for Phase 2. | Source artefacts ready; awaiting Phase 2 ntro-mcp + adapter (N-76) |

## Adding a new agent

1. Create `agents/<name>/` with:
   - `agent.yaml` — Anthropic API-shaped metadata (`name`, `model`, `system`, `tools`, `mcp_servers`, `skills`, `metadata`)
   - `system-prompt.md` — the system message
   - `tools.md` — toolset + MCP server + skills reference
   - `*.md` skeletons or templates the agent fills in (e.g. `handover-template.md`)
   - `README.md` — provisioning + registration playbook for this specific agent
2. Make sure the agent's process loop calls **MCP tools** for auth-required operations (`ntro_steps_*`, `ntro_tasks_*`). The Ntropii API key never appears in the agent's code or env — Anthropic's runtime injects it via the Vault.
3. Provision via Anthropic's Console (or `POST /v1/agents`); register the deployed agent_id with Ntropii via `ntro agent create --path anthropic://agents/<id> --tenant <slug>`.

## License / sharing

This repo is intended as the canonical Claude-side spec for Ntropii integration. Anthropic Managed Agents that need to call into Ntropii should reference the docs and example here.
