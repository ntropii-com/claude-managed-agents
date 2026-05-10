# claude-managed-agents

Claude-specific reference + source artefacts for **Anthropic Managed Agents that integrate with Ntropii**.

The repo answers a single question: *"I'm building a Claude MA that needs to call into Ntropii — how does that work?"*

It contains:

| Path | Purpose |
|---|---|
| [`docs/integration-spec.md`](docs/integration-spec.md) | The contract: env vars, lifecycle, file passback, error semantics |
| [`docs/sdk-reference.md`](docs/sdk-reference.md) | The `ntro.workflow.*` SDK surface a Claude MA `import`s |
| [`agents/<name>/`](agents) | Source artefacts for a specific reference agent (system prompt, tools config, agent.yaml, README) |

## Scope

- **Claude Managed Agents only.** Other ecosystems (OpenAI Assistants, Vertex AI Agent Builder, Azure AI Agent Service) integrate via the same `ntro.workflow.*` SDK contract but use their respective SDKs/runtimes; their reference artefacts will live in sibling repos when they're built.
- **Single-tenant agent code.** Tenant-specific configuration (entity ids, period selectors, output filenames) is passed in at Run time via `ntro.workflow.tasks.get_period()` and friends. Agents themselves are tenant-neutral.

## Status

The SDK surface (`ntro.workflow.steps.*`, `ntro.workflow.tasks.*`) is Phase 2 work, tracked as [Linear N-76](https://linear.app/byng/issue/N-76). It is not yet in the published `ntro` package on PyPI; agents in this repo register cleanly on Anthropic today but cannot complete a Run end-to-end until N-76 ships and `ntro` 0.2.0 is published.

The integration contract (env vars, lifecycle, file passback) is stable — what's documented in `docs/integration-spec.md` is what Phase 2 will build to.

## Reference agents

| Agent | Description | Status |
|---|---|---|
| [`agents/audit-handover/`](agents/audit-handover) | Produces an auditor handover .docx for a completed NAV close. First demo agent for Phase 2. | Source artefacts ready; awaiting SDK + adapter (N-76) |

## Adding a new agent

1. Create `agents/<name>/` with these files (model the audit-handover example):
   - `agent.yaml` — Anthropic API-shaped metadata (`name`, `model`, `system`, `tools`, `execution_environment`)
   - `system-prompt.md` — paste into the Console's "System message" field, or inline into `agent.yaml.system` for API submission
   - `tools.md` — toolset + pip + env reference
   - `handover-template.md` (or analogous) — any markdown skeletons the agent fills in
   - `README.md` — provisioning + registration playbook for this specific agent
2. Make sure the agent's process loop only calls `ntro.workflow.*` SDK methods documented in [`docs/sdk-reference.md`](docs/sdk-reference.md). Don't reach for Anthropic-tool-specific paths (e.g. native docx skill) — they don't exist in Managed Agents' unified toolset.
3. Provision via the Anthropic Console (no dedicated CLI for Managed Agents); register the deployed agent_id with Ntropii via `ntro agent create --path anthropic://agents/<id> --tenant <slug>`.

## License / sharing

This repo is intended as the canonical Claude-side spec for Ntropii integration. Anthropic Managed Agents that need to call into Ntropii should reference the docs and example here.
