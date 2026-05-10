# audit-handover Claude MA — provisioning + registration

Source artefacts for the `audit-handover` Claude Managed Agent — produces a Ntropii-style auditor handover .docx for a completed NAV close.

## ⚠ Architectural blocker (open)

The `POST /v1/agents` create body for Managed Agents has only these fields: `name`, `model`, `system`, `tools`, `description`, `mcp_servers`, `skills`, `metadata`. **There is no first-class field for**:

- **Pip dependencies** (so `ntro` can be installed in the agent's Python sandbox)
- **Environment variables** (so `NTRO_API_KEY` / `NTRO_TENANT_URL` can authenticate the SDK)

Phase 2 / [N-76](https://linear.app/byng/issue/N-76) was scoped assuming both. They aren't there. Until that's resolved, the agent can be **registered** on Anthropic + Ntropii, but `import ntro` inside the agent's `bash python …` will fail (no `ntro` in the sandbox) — i.e. the agent won't actually be functional. Path-forward options under investigation:

1. **MCP server bridge** — Ntropii ships a public MCP server with the same tool surface as `ntro.workflow.steps.*` / `tasks.*`; the agent talks to it via `mcp_servers` (which IS a first-class field). Drops the SDK-over-pip approach but keeps the same wire protocol. Probably the right path.
2. **Anthropic Skills** — investigate whether `skills` can carry pip deps + env. Likely not, but worth a 30-min look.
3. **Wait for Anthropic** — Managed Agents is in beta; pip + env may land. Not a path we can ship on.

The Phase 2 framing is being revised toward (1) — see N-76 description for the updated approach.

## What's in this folder

| File | Purpose |
|---|---|
| `agent.yaml` | Anthropic API-shaped metadata. Matches `POST /v1/agents` body fields exactly. |
| `system-prompt.md` | The agent's system message (paste into `agent.yaml.system` or the Console form) |
| `tools.md` | Toolset config notes |
| `handover-template.md` | Markdown skeleton the agent fills in |
| `README.md` | This file |

## Provisioning the agent (despite the blocker above)

You can still register the agent shell on Anthropic to validate the rest of the flow:

### Console path

1. Open <https://console.anthropic.com> → Agents → Create.
2. **Name:** `audit-handover`
3. **Description:** copy from `agent.yaml.description`
4. **Model:** `claude-sonnet-4-6`
5. **System message:** paste the full contents of `system-prompt.md`
6. **Toolset:** `agent_toolset_20260401` — enable `bash`, `write`, `read`
7. Capture the resulting `agent_id` (e.g. `agent_01HX…`)

### API path (curl)

```bash
SYSTEM_PROMPT=$(cat system-prompt.md)

curl -sS https://api.anthropic.com/v1/agents \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d "$(jq -n \
        --arg name        "audit-handover" \
        --arg model       "claude-sonnet-4-6" \
        --arg system      "$SYSTEM_PROMPT" \
        --arg description "Produces an auditor handover .docx for a completed NAV close." \
        --argjson tools '[{
          "type":"agent_toolset_20260401",
          "configs":[
            {"name":"bash","enabled":true},
            {"name":"write","enabled":true},
            {"name":"read","enabled":true}
          ]
        }]' \
        '{name:$name, model:$model, system:$system, description:$description, tools:$tools}')" \
  | jq -r '.id // .error // .'
```

## Register the agent with Ntropii (when N-76 ships)

```bash
ntro agent create \
  --path "anthropic://agents/<agent_id>" \
  --tenant byng \
  --name audit-handover
```

This returns an Ntropii API key. With Phase 2's MCP-bridge approach, that key gets configured on the agent's `mcp_servers[].auth` rather than as an env var.

## What works today vs what waits for N-76

| Capability | Today | After N-76 |
|---|---|---|
| Register agent shell on Anthropic | ✓ | ✓ |
| Register agent with Ntropii (`ntro agent create`) | ✗ (CLI subcommand not wired) | ✓ |
| Agent invoked from a runbook step | ✗ (no `ntro.workflow.agents.invoke`) | ✓ |
| Agent calls back into Ntropii | ✗ (no SDK / MCP bridge) | ✓ |
| Run-output file passback | ✗ (no adapter) | ✓ |
| Breadcrumb child steps from agent | ✗ (no StepEvent ingress) | ✓ |
