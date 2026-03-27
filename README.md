# Langfuse MCP Server (Multi-Environment Fork)

[![Python 3.10–3.13](https://img.shields.io/badge/python-3.10–3.13-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Fork of [avivsinai/langfuse-mcp](https://github.com/avivsinai/langfuse-mcp) (v0.5.2) with **multi-environment support**. Switch between dev, prod, and local Langfuse instances using a single `env` parameter on every tool.

## Multi-Environment Setup

### 1. Create `config.json`

Copy `config.example.json` and fill in your credentials:

```json
{
  "default_env": "dev",
  "environments": {
    "dev": {
      "host": "http://your-dev-langfuse:3000",
      "public_key": "pk-lf-...",
      "secret_key": "sk-lf-..."
    },
    "prod": {
      "host": "http://your-prod-langfuse:3000",
      "public_key": "pk-lf-...",
      "secret_key": "sk-lf-..."
    },
    "local": {
      "host": "http://localhost:3000",
      "public_key": "pk-lf-...",
      "secret_key": "sk-lf-..."
    }
  }
}
```

`config.json` is gitignored. Path configurable via `LANGFUSE_MCP_CONFIG` env var.

### 2. Register with Claude Code

#### Option A: Multi-environment (with `config.json`)

```bash
# User-scoped (available in all projects)
claude mcp add --transport stdio --scope user \
  --env LANGFUSE_MCP_CONFIG=/absolute/path/to/langfuse-mcp/config.json \
  langfuse \
  -- /absolute/path/to/langfuse-mcp/.venv/bin/langfuse-mcp

# Project-scoped (shared via .mcp.json checked into git)
claude mcp add --transport stdio --scope project \
  --env LANGFUSE_MCP_CONFIG=/absolute/path/to/langfuse-mcp/config.json \
  langfuse \
  -- /absolute/path/to/langfuse-mcp/.venv/bin/langfuse-mcp
```

#### Option B: Single-environment (env vars only, no config.json)

```bash
claude mcp add --transport stdio --scope user \
  --env LANGFUSE_PUBLIC_KEY=pk-lf-... \
  --env LANGFUSE_SECRET_KEY=sk-lf-... \
  --env LANGFUSE_HOST=https://cloud.langfuse.com \
  langfuse \
  -- /absolute/path/to/langfuse-mcp/.venv/bin/langfuse-mcp
```

#### Optional flags

Append these after the `--` separator to customize behavior:

```bash
  -- /path/to/.venv/bin/langfuse-mcp \
  --tools traces,observations,sessions,exceptions,prompts \  # load specific tool groups
  --read-only                                                 # disable write operations
```

#### Verify it works

```bash
claude mcp list          # confirm "langfuse" appears
```

Inside a Claude Code session, run `/mcp` to check server status and available tools.

### 3. Use the `env` parameter

Every tool accepts an optional `env` parameter:

```
# Uses default_env (dev)
fetch_traces(age=60)

# Explicitly target prod
fetch_traces(age=60, env="prod")

# Query local instance
fetch_trace(trace_id="abc123", env="local")
```

### Backward Compatibility

Without `config.json`, the server falls back to standard single-environment mode using `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, and `LANGFUSE_HOST` env vars or CLI args — identical to the upstream behavior.

## Tools (25 total)

| Category | Tools |
|----------|-------|
| Traces | `fetch_traces`, `fetch_trace` |
| Observations | `fetch_observations`, `fetch_observation` |
| Sessions | `fetch_sessions`, `get_session_details`, `get_user_sessions` |
| Exceptions | `find_exceptions`, `find_exceptions_in_file`, `get_exception_details`, `get_error_count` |
| Prompts | `list_prompts`, `get_prompt`, `get_prompt_unresolved`, `create_text_prompt`, `create_chat_prompt`, `update_prompt_labels` |
| Datasets | `list_datasets`, `get_dataset`, `list_dataset_items`, `get_dataset_item`, `create_dataset`, `create_dataset_item`, `delete_dataset_item` |
| Schema | `get_data_schema` |

## Dataset Item Updates (Upsert)

Langfuse uses upsert for dataset items. To edit an existing item, call `create_dataset_item` with `item_id`. If the ID exists, it updates; otherwise it creates a new item.

```python
create_dataset_item(
  dataset_name="qa-test-cases",
  item_id="item_123",
  input={"question": "What is 2+2?"},
  expected_output={"answer": "4"}
)
```

## Skill

This project includes a skill with debugging playbooks.

**Via [skills](https://github.com/vercel-labs/add-skill)** (recommended):
```bash
npx skills add avivsinai/langfuse-mcp -g -y
```

**Via [skild](https://skild.sh)**:
```bash
npx skild install @avivsinai/langfuse -t claude -y
```

**Manual install:**
```bash
cp -r skills/langfuse ~/.claude/skills/   # Claude Code
cp -r skills/langfuse ~/.codex/skills/    # Codex CLI
```

Try asking: "help me debug langfuse traces"

See [`skills/langfuse/SKILL.md`](skills/langfuse/SKILL.md) for full documentation.

## Selective Tool Loading

Load only the tool groups you need to reduce token overhead:

```bash
langfuse-mcp --tools traces,prompts
```

Available groups: `traces`, `observations`, `sessions`, `exceptions`, `prompts`, `datasets`, `schema`

## Read-Only Mode

Disable all write operations for safer read-only access:

```bash
langfuse-mcp --read-only
# Or via environment variable
LANGFUSE_MCP_READ_ONLY=true langfuse-mcp
```

This disables: `create_text_prompt`, `create_chat_prompt`, `update_prompt_labels`, `create_dataset`, `create_dataset_item`, `delete_dataset_item`

## Other Clients

### Cursor

Create `.cursor/mcp.json` in your project (or `~/.cursor/mcp.json` for global):

```json
{
  "mcpServers": {
    "langfuse": {
      "command": "uvx",
      "args": ["--python", "3.11", "langfuse-mcp"],
      "env": {
        "LANGFUSE_PUBLIC_KEY": "pk-...",
        "LANGFUSE_SECRET_KEY": "sk-...",
        "LANGFUSE_HOST": "https://cloud.langfuse.com"
      }
    }
  }
}
```

### Docker

```bash
docker run --rm -i \
  -e LANGFUSE_PUBLIC_KEY=pk-... \
  -e LANGFUSE_SECRET_KEY=sk-... \
  -e LANGFUSE_HOST=https://cloud.langfuse.com \
  ghcr.io/avivsinai/langfuse-mcp:latest
```

## Development

```bash
uv venv --python 3.11 .venv && source .venv/bin/activate
uv pip install -e ".[dev]"
pytest
```

## License

MIT
