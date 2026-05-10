# Tools enabled

Anthropic Managed Agents use a single unified toolset (`agent_toolset_20260401`), not the per-tool `type` enums you see on the Messages API. There's no native Python sandbox tool and no native docx skill — the agent uses `bash` to run Python and `python-docx` to render the final document.

## Toolset configuration (Console "Tools" tab)

Within `agent_toolset_20260401`, enable:

| Tool | Why |
|---|---|
| `bash` | Invokes `python …` to run scripts the agent writes. Also lets the agent install any missing pip deps if needed. |
| `write` | Writes the agent's Python scripts (and the markdown draft) to local files in the sandbox so they can be `bash`-executed. |
| `read` | Reads intermediate files between steps (the markdown draft, the period summary, etc.). |

Disable everything else — `edit`, `glob`, `grep`, `web_fetch`, `web_search` are not needed for this PoC and add latency / risk.

## Pip dependencies (Console "Python execution" tab)

| Package | Why |
|---|---|
| `ntro>=0.1.0,<0.2.0` | The runtime SDK — `ntro.workflow.steps.*`, `ntro.workflow.tasks.*`. |
| `python-docx>=1.1.0` | Used to render the final `.docx`. The Ntropii adapter pulls files attached to the Run output at completion and persists them to the tenant data plane. |

## Environment variables (Console "Environment" tab)

Set these so the SDK can authenticate. The Ntropii adapter sets `NTRO_SESSION_ID` per-Run; the others are filled in once after Ntropii registration (see README.md).

- `NTRO_API_KEY`
- `NTRO_TENANT_URL`
- `NTRO_SESSION_ID`
