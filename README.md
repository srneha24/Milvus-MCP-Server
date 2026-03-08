# Milvus MCP Server (Read-Only Fork)

This repository is a personal fork of the official [`zilliztech/mcp-server-milvus`](https://github.com/zilliztech/mcp-server-milvus), customized to expose **read-only and inspection-focused MCP tools**.

It provides Model Context Protocol (MCP) access to Milvus for querying, searching, and metadata inspection, without schema/data mutation tools such as create/insert/delete.

## Features

- Read-focused Milvus operations for LLM assistants
- Text, vector, hybrid, and text-similarity search tools
- Collection/database inspection and database switching
- `stdio` (default) and `sse` transport modes
- Environment-variable based configuration

## Available Tools

- `milvus_list_collections`
- `milvus_get_collection_info`
- `milvus_query`
- `milvus_text_search`
- `milvus_vector_search`
- `milvus_hybrid_search`
- `milvus_text_similarity_search` (Milvus 2.6.0+ with embedding function configured)
- `milvus_load_collection`
- `milvus_release_collection`
- `milvus_list_databases`
- `milvus_use_database`

## Prerequisites

- Python 3.10+
- A running Milvus instance (local or remote)
- `uv` installed

## Setup

```bash
git clone https://github.com/srneha24/Milvus-MCP-Server.git
cd Milvus-MCP-Server
uv sync
```

Optional local config (`src/mcp_server_milvus/.env`):

```env
MILVUS_URI="http://localhost:19530"
MILVUS_TOKEN=""
MILVUS_DB="default"
```

`MILVUS_*` values from `.env` or environment variables override CLI flags.

## Run Locally

### Stdio mode (default)

```bash
uv run src/mcp_server_milvus/server.py --milvus-uri http://localhost:19530
```

### SSE mode

```bash
uv run src/mcp_server_milvus/server.py --sse --milvus-uri http://localhost:19530 --port 8000
```

## Server Arguments (`parse_arguments`)

The server accepts these CLI flags in `src/mcp_server_milvus/server.py`:

- `--milvus-uri` (default: `http://localhost:19530`)
  - Milvus endpoint to connect to.
  - Use in local runs or in stdio client configs when not setting `MILVUS_URI`.
- `--milvus-token` (default: `None`)
  - Auth token for secured Milvus deployments.
  - Use for cloud/secured Milvus; for local unauthenticated setups, leave unset.
- `--milvus-db` (default: `default`)
  - Initial database name selected at startup.
  - Use when you want a non-default Milvus database without calling `milvus_use_database` later.
- `--sse` (flag, default: off)
  - Enables SSE transport mode instead of stdio.
  - Required when exposing the server via URL (for Claude Desktop/Cursor/Codex SSE configs).
- `--port` (default: `8000`)
  - HTTP port used only with `--sse`.
  - Change it when `8000` is occupied or when running multiple MCP servers.

Examples:

```bash
# Stdio with explicit DB and token
uv run src/mcp_server_milvus/server.py --milvus-uri http://localhost:19530 --milvus-token <token> --milvus-db mydb

# SSE on a custom port
uv run src/mcp_server_milvus/server.py --sse --milvus-uri http://localhost:19530 --milvus-db analytics --port 9000
```

Note: `MILVUS_URI`, `MILVUS_TOKEN`, and `MILVUS_DB` from environment variables or `.env` override CLI flag values.

## Usage with Claude Desktop

Claude Desktop config file:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\\Claude\\claude_desktop_config.json`

### Stdio configuration

```json
{
  "mcpServers": {
    "milvus": {
      "command": "uv",
      "args": [
        "--directory",
        "/ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus",
        "run",
        "server.py"
      ],
      "env": {
        "MILVUS_URI": "http://localhost:19530",
        "MILVUS_TOKEN": "",
        "MILVUS_DB": "default"
      }
    }
  }
}
```

### SSE configuration

Start the server first:

```bash
uv run src/mcp_server_milvus/server.py --sse --milvus-uri http://localhost:19530 --port 8000
```

Then configure Claude Desktop with an SSE endpoint:

```json
{
  "mcpServers": {
    "milvus-sse": {
      "url": "http://127.0.0.1:8000/sse",
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

## Usage with Claude Code

### Stdio

```bash
claude mcp add milvus --transport stdio --env MILVUS_URI=http://localhost:19530 --env MILVUS_TOKEN= --env MILVUS_DB=default -- uv --directory /ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus run server.py
```

Milvus Cloud example (token required):

```bash
claude mcp add milvus --transport stdio --env MILVUS_URI=https://in03-xxxx.api.gcp-us-west1.zillizcloud.com --env MILVUS_TOKEN=<MILVUS_CLOUD_TOKEN> --env MILVUS_DB=default -- uv --directory /ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus run server.py
```

Project-local install:

```bash
claude mcp add milvus --transport stdio --env MILVUS_URI=http://localhost:19530 --env MILVUS_TOKEN= --env MILVUS_DB=default --scope local -- uv --directory /ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus run server.py
```

### SSE

Start the server first:

```bash
uv run src/mcp_server_milvus/server.py --sse --milvus-uri http://localhost:19530 --port 8000
```

Milvus Cloud example (token required):

```bash
MILVUS_URI=https://in03-xxxx.api.gcp-us-west1.zillizcloud.com MILVUS_TOKEN=<MILVUS_CLOUD_TOKEN> uv run src/mcp_server_milvus/server.py --sse --milvus-db default --port 8000
```

PowerShell equivalent:

```powershell
$env:MILVUS_URI="https://in03-xxxx.api.gcp-us-west1.zillizcloud.com"
$env:MILVUS_TOKEN="<MILVUS_CLOUD_TOKEN>"
uv run src/mcp_server_milvus/server.py --sse --milvus-db default --port 8000
```

Then add the SSE endpoint:

```bash
claude mcp add --transport sse milvus-sse http://127.0.0.1:8000/sse
```

## Usage with OpenAI Codex

### Stdio

```bash
codex mcp add milvus --env MILVUS_URI=http://localhost:19530 --env MILVUS_TOKEN= --env MILVUS_DB=default -- uv --directory /ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus run server.py
```

Milvus Cloud example (token required):

```bash
codex mcp add milvus --env MILVUS_URI=https://in03-xxxx.api.gcp-us-west1.zillizcloud.com --env MILVUS_TOKEN=<MILVUS_CLOUD_TOKEN> --env MILVUS_DB=default -- uv --directory /ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus run server.py
```

Or add to `~/.codex/config.toml` (or project `.codex/config.toml`):

```toml
[mcp_servers.milvus]
command = "uv"
args = ["--directory", "/ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus", "run", "server.py"]

[mcp_servers.milvus.env]
MILVUS_URI = "http://localhost:19530"
MILVUS_TOKEN = ""
MILVUS_DB = "default"
```

### SSE

Start the server first:

```bash
uv run src/mcp_server_milvus/server.py --sse --milvus-uri http://localhost:19530 --port 8000
```

Milvus Cloud example (token required):

```bash
MILVUS_URI=https://in03-xxxx.api.gcp-us-west1.zillizcloud.com MILVUS_TOKEN=<MILVUS_CLOUD_TOKEN> uv run src/mcp_server_milvus/server.py --sse --milvus-db default --port 8000
```

PowerShell equivalent:

```powershell
$env:MILVUS_URI="https://in03-xxxx.api.gcp-us-west1.zillizcloud.com"
$env:MILVUS_TOKEN="<MILVUS_CLOUD_TOKEN>"
uv run src/mcp_server_milvus/server.py --sse --milvus-db default --port 8000
```

Then register the endpoint URL in Codex:

```bash
codex mcp add milvus-sse --url http://127.0.0.1:8000/sse
```

Or configure it directly in `~/.codex/config.toml`:

```toml
[mcp_servers.milvus_sse]
url = "http://127.0.0.1:8000/sse"
```

## Usage with Cursor

Cursor supports MCP via `mcp.json`.

### Stdio example

```json
{
  "mcpServers": {
    "milvus": {
      "command": "uv",
      "args": [
        "--directory",
        "/ABSOLUTE/PATH/TO/Milvus/src/mcp_server_milvus",
        "run",
        "server.py"
      ]
    }
  }
}
```

### SSE example

Start the server first:

```bash
uv run src/mcp_server_milvus/server.py --sse --milvus-uri http://localhost:19530 --port 8000
```

Then point Cursor to the SSE endpoint:

```json
{
  "mcpServers": {
    "milvus-sse": {
      "url": "http://127.0.0.1:8000/sse",
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

## Troubleshooting

- Connection errors: verify Milvus is running and URI/port are correct.
- Auth errors: verify `MILVUS_TOKEN` and server-side auth settings.
- Tools missing in client: restart the MCP client and check server logs.
- For local dev, try `127.0.0.1` instead of `localhost` if DNS/loopback resolution is inconsistent.
