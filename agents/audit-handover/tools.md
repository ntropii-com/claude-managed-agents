# Tools, MCP, and Skills — Console reference

Three Console tabs cover the agent's runtime surface: **Tools**, **MCP servers**, **Skills**. Each maps directly to a top-level field on the `POST /v1/agents` body.

## Tools tab

Anthropic Managed Agents use a single unified toolset (`agent_toolset_20260401`). Within it, enable only what the agent needs:

| Tool | Why |
|---|---|
| `bash` | Invokes `python …` to run scripts the agent writes. |
| `write` | Writes the agent's Python scripts (and the markdown draft) to `/tmp/` so they can be `bash`-executed. |
| `read` | Reads intermediate files between steps. |

In the same Tools tab, ALSO add an **MCP toolset** entry pointing at the Ntropii MCP server (declared in the next tab):

```yaml
- type: mcp_toolset
  mcp_server_name: ntropii
```

(In the API body this lives in the same `tools` array as the toolset above.)

Disable everything else — `edit`, `glob`, `grep`, `web_fetch`, `web_search` are not needed for this PoC and add latency / risk.

## MCP servers tab

Add **one** entry pointing at the Ntropii MCP server:

| Field | Value |
|---|---|
| **Type** | `url` |
| **Name** | `ntropii` (must match `mcp_server_name` in the Tools tab) |
| **URL** | `https://mcp.test.ntropii.com/v1` (test cluster ntro-mcp) |

Auth is **not** configured here. The Vault attached at session start carries the Ntropii API key, and Anthropic's runtime injects it into MCP server requests automatically.

API-body equivalent:

```json
{
  "mcp_servers": [
    {"type": "url", "name": "ntropii", "url": "https://mcp.test.ntropii.com/v1"}
  ]
}
```

## Skills tab

Optional but recommended: enable Anthropic's pre-built **`docx`** skill. It auto-invokes when the agent needs to produce a Word document, and can be a richer alternative to direct `python-docx` authoring.

| Field | Value |
|---|---|
| **Type** | `anthropic` (Anthropic-built) |
| **Skill ID** | `docx` |

API-body equivalent:

```json
{
  "skills": [
    {"type": "anthropic", "skill_id": "docx"}
  ]
}
```

You can layer additional skills (`xlsx`, `pdf`) if a future agent needs them; max 20 skills per session.

For **custom** skills (Anthropic's "Agent Skills" — domain-specific expertise packs uploaded as filesystems), use `{type: "custom", skill_id: "skill_abc", version: "latest"}`. The audit-handover agent doesn't need any custom skills today.

## Pip dependencies — Environment, NOT Agent

The Console field for pip packages lives on the **Environment** object (separate from the Agent). See [README.md → Setup checklist → step 2](./README.md#2-environment--pip-packages--networking).

For this agent, the Environment's `pip` packages are:

```
ntro python-docx
```

(or pinned: `ntro==0.1.0 python-docx==1.1.0`)

## Environment variables — Vault, NOT direct env vars

There are no environment variables in Managed Agents. The Ntropii API key lives in a **Vault** referenced at session start; Anthropic's runtime injects it into MCP server auth headers, so it never appears in the agent's bash sandbox or system prompt.

Set up the Vault once on Anthropic; reuse it across sessions and agents.
