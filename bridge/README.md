# MCP VoidEditor Bridge — README

> **Language**: [🇺🇸 English](README.md) | [🇮🇩 Bahasa Indonesia](docs_id/README.md)

**Summary**  
This project is a _starter_ for creating an **MCP bridge** that enables **sub-agents** to operate inside **VoidEditor** with full IDE capabilities, just like agents work in Cursor.

**Concept**: Think of Cursor as a **Big City** where your main agent lives, and VoidEditor as a **Small City** inside the big city where sub-agents can work independently. The bridge approach keeps VoidEditor largely unmodified, provides permission control, and facilitates auditing.

> All components are built for **lab / sandbox**. Apply hardening before production.

---

## Components

1. **Bridge (FastAPI)** — `bridge_app.py`
   - HTTP endpoints for: request access, approve/deny, session management.
   - WebSocket for agent (session channel).
   - WebSocket or SSE for notifications to Cursor UI.
2. **VoidEditor Plugin (Sub-Agent Runtime)** — `plugin_void.js`
   - **Full IDE Environment**: Provides complete development environment for sub-agents.
   - Connects to bridge (plugin WS) and executes full IDE operations.
   - Handles: file ops, code analysis, terminal, search, git, project management.
   - Real-time state sync: cursor, selection, viewport, buffers.
   - If VoidEditor doesn't have an API, plugin works through filesystem/CLI hooks.
3. **Cursor (client)** — MCP client configuration in Cursor to call `POST /mcp/request_access` and display consent UI.

---

## Sub-Agent Features Available

**Core IDE Operations:**

- **File Management**: Create, read, edit, delete, rename files
- **Code Analysis**: Syntax analysis, error detection, code completion, formatting
- **Project Operations**: Explore project tree, build, run tests, workspace management
- **Terminal Integration**: Execute commands, manage terminal sessions
- **Version Control**: Git operations (status, commit, branch, diff)
- **Search & Replace**: Find in files, project-wide search, regex operations
- **Real-time Sync**: Cursor position, selection, viewport, buffer changes

**Session Management:**

- Request access → user approve/deny via Cursor UI.
- Ephemeral session: token TTL (default 5 minutes).
- WebSocket session proxy: sub-agent ↔ bridge ↔ plugin.
- Granular permissions with scope-based access control.
- Root path restriction (ALLOWED_ROOT).

---

## Requirements (development)

- Python 3.10+
- Node.js 18+
- pip packages: `fastapi uvicorn python-multipart`
- npm packages: `ws` (or according to plugin)

---

## Environment configuration

Set environment variables (or use `.env`):

- `MCP_TOKEN` — secret token for management endpoints (Cursor → bridge). **Don't commit**.
- `ALLOWED_ROOT` — root workspace (example: `/home/minixo/lab/projects`)
- `HOST` — default `127.0.0.1`
- `PORT` — default `8787`
- `SESSION_TTL` — default `300` (seconds)

---

## Install & Run (example)

1. Clone starter repo (e.g. `starter-mcp-void`).
2. Python virtual env:

```bash
python -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn
```

3. Export env vars:

```bash
export MCP_TOKEN="dev-token"
export ALLOWED_ROOT="${HOME}/lab/projects"
```

4. Run bridge:

```bash
uvicorn bridge_app:app --host 127.0.0.1 --port 8787 --reload
```

5. Run VoidEditor plugin (example Node script):

```bash
node plugin_void.js
```

---

## Example flow - Sub-Agent Development

1. Main Agent (Cursor) requests sub-agent access to VoidEditor:

```bash
curl -X POST "http://127.0.0.1:8787/mcp/request_access" \
  -H "Authorization: ${MCP_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id":"sub-agent-dev-1",
    "scopes":["read:files","edit:buffers","analyze:syntax","build:project","git:commit"],
    "roots":["/home/minixo/lab/projects/myrepo"],
    "reason":"Sub-agent development workflow in VoidEditor"
  }'
```

2. Cursor UI receives `PENDING` notification (via SSE/WS), user clicks **Approve**. Cursor calls:

```bash
curl -X POST "http://127.0.0.1:8787/mcp/approve" \
  -H "Authorization: ${MCP_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"request_id":"<id>","ttl_seconds":300}'
```

Response contains `session_id` & `session_token`.

3. Agent connects to WebSocket:

```text
ws://127.0.0.1:8787/mcp/session/<session_id>
Header: Authorization: Bearer <session_token>
```

4. Sub-Agent performs full development workflow:

```json
// Explore project structure
{
  "type": "cmd",
  "id": "uuid-1",
  "action": "explore_tree",
  "args": { "path": "myrepo" }
}

// Analyze code for issues
{
  "type": "cmd",
  "id": "uuid-2",
  "action": "analyze_syntax",
  "args": { "file": "myrepo/main.py" }
}

// Edit code with improvements
{
  "type": "cmd",
  "id": "uuid-3",
  "action": "edit_buffer",
  "args": { "file": "main.py", "changes": "refactored_code" }
}

// Run tests to verify changes
{
  "type": "cmd",
  "id": "uuid-4",
  "action": "exec_command",
  "args": { "command": "python -m pytest" }
}
```

Plugin in VoidEditor executes each command and provides real-time feedback to sub-agent.

---

## Policy & security (important)

- Bridge must run on **localhost** or Unix socket.
- Restrict `ALLOWED_ROOT`.
- Session tokens are ephemeral (TTL).
- Limit edit size (`max_edit_bytes`) & rate limit.
- Audit all requests (JSON log).
- Destructive actions require explicit confirmation UI.
- Monitor plugin for input validation (sanitize paths, no shell injection).

---

## Cursor integration

Cursor needs to:

- Call `POST /mcp/request_access` when agent needs access.
- Listen to notifications (SSE/WS) to display **Consent** modal.
- Call `POST /mcp/approve` or `POST /mcp/deny`.
- Display session status (time left) & revoke button.

---

## Logs & debugging

- Bridge prints audit JSON lines to `./logs/audit.log`.
- Plugin prints events to console.
- Use Wireshark/tcpdump only in lab to inspect WebSocket traffic if needed.

---

## Next steps (optional)

- Implement plugin using VoidEditor internal API (if available).
- Add persistent storage for requests & audit (sqlite).
- Add TLS / mTLS if crossing host boundary.
- Implement CRDT/OT for multi-editor collaborative sessions (advanced).

---

## Documentation

- **[BLUEPRINT.md](BLUEPRINT.md)** — Complete technical design (API, message schema, lifecycle, security policies, testing checklist, and integration options to Cursor & VoidEditor)
- **[docs_id/](docs_id/)** — Indonesian documentation

---

**Need implementation files?** Check [BLUEPRINT.md](BLUEPRINT.md) for complete technical specifications, or see the Indonesian version at [docs_id/BLUEPRINT.md](docs_id/BLUEPRINT.md).
