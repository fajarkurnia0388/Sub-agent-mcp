# MCP Bridge — BLUEPRINT (Technical Design)

> **Language**: [🇺🇸 English](BLUEPRINT.md) | [🇮🇩 Bahasa Indonesia](docs_id/BLUEPRINT.md)

This document is a technical blueprint for **MCP Bridge** that enables **sub-agents** to operate inside VoidEditor with full IDE capabilities, just like agents work in Cursor.

**Concept**: Cursor = Main Agent environment, VoidEditor = Extended environment for sub-agents.

Focus: architecture, API spec, WS messages, lifecycle, security, testing, and edge-cases.

---

## Objectives

Provide a secure and auditable mediator that:

- receives access requests from agents in Cursor,
- requests user approval through Cursor UI,
- creates ephemeral sessions for **sub-agents** to operate inside VoidEditor,
- enables sub-agents to perform full IDE operations (like in Cursor) within VoidEditor scope,
- forwards commands with enforcement policy (roots/scopes/limits).

---

## Components & Responsibilities

1. **Cursor (MCP client & UI)**

   - Send `request_access`.
   - Display approval modal.
   - Call `approve/deny`.
   - Display session status & revoke control.

2. **Bridge (FastAPI)**

   - Management endpoints: request, list, approve, deny, revoke.
   - `events` stream to Cursor (SSE or WS).
   - WebSocket server for agent session (`/mcp/session/{session_id}`).
   - WebSocket client/server for plugin (`/plugin/{plugin_id}`) or connector to plugin socket.
   - Policy enforcement, audit logs, session lifecycle.

3. **VoidEditor Plugin (Sub-Agent Runtime)**
   - Connected to bridge (persistent WS).
   - **Full IDE Capabilities**: Provides complete development environment to sub-agents.
   - Handle actions: file ops, code analysis, terminal, search, git, project management, debugging.
   - Real-time state sync: cursor, selection, viewport, buffers, project state.
   - Emit events: all editor state changes, user interactions, system events, code analysis results.

---

## Data model (in-memory minimal)

- `REQUEST`:

  - `request_id: uuid`
  - `agent_id: str`
  - `scopes: [str]`
  - `roots: [str]`
  - `reason: str`
  - `status: ENUM(PENDING, APPROVED, DENIED)`
  - `created_at: timestamp`
  - `approved_by: str | null`
  - `session_id: str | null`

- `SESSION`:

  - `session_id: uuid`
  - `token: uuid`
  - `expires_at: timestamp`
  - `allowed_scopes: [str]`
  - `roots: [str]`
  - `ws_agent: ws_connection`
  - `ws_plugin: ws_connection` (optional, bridge proxies to plugin)
  - `created_by_request: request_id`

- `AUDIT_LOG`:
  - JSON lines: `{ts, actor (agent_id/user), action, args, result, session_id, request_id}`

---

## API Specification (HTTP)

All management endpoints require header `Authorization: ${MCP_TOKEN}` (management secret). Cursor uses this to approve/deny.

1. `POST /mcp/request_access`
   - Body:

```json
{
  "agent_id": "string",
  "scopes": ["read:buffers", "open:file", "edit:buffer"],
  "roots": ["/home/..."],
  "reason": "string"
}
```

- Response: `{"request_id":"uuid"}`

2. `GET /mcp/requests`

   - Return list of recent requests.

3. `POST /mcp/approve`
   - Body:

```json
{
  "request_id": "uuid",
  "approved_scopes": ["read:buffers", "open:file"],
  "ttl_seconds": 300
}
```

- Response:

```json
{"session_id":"uuid","session_token":"uuid","expires_at":<ts>}
```

4. `POST /mcp/deny`

   - Body: `{ "request_id":"uuid" }`

5. `POST /mcp/revoke`

   - Body: `{ "session_id":"uuid" }`
   - Force close session.

6. `GET /mcp/events` (SSE) or WebSocket `/mcp/events/ws`
   - Real-time notifications for Cursor UI about new requests & session changes.

---

## WebSocket Session Spec (agent <-> bridge)

- Endpoint: `ws://HOST:PORT/mcp/session/{session_id}`
- Header: `Authorization: Bearer <session_token>`
- Message format JSON.

### Commands (sub-agent → bridge/plugin)

Sub-agents can perform full IDE operations like in Cursor:

- `cmd`:

```json
{
  "type":"cmd",
  "id":"uuid",
  "action": "file_operation" | "buffer_management" | "code_analysis" |
           "project_management" | "terminal_execution" | "version_control" |
           "search_replace" | "realtime_sync" | "advanced_features",
  "args": { ... }
}
```

**Action Categories**:

- **File Ops**: `open_file`, `create_file`, `delete_file`, `rename_file`, `search_files`, `watch_file`
- **Buffer Mgmt**: `get_buffers`, `edit_buffer`, `switch_buffer`, `split_pane`, `close_buffer`
- **Code Analysis**: `analyze_syntax`, `get_errors`, `get_completions`, `format_code`, `goto_definition`
- **Project**: `explore_tree`, `build_project`, `run_tests`, `get_workspace_info`
- **Terminal**: `exec_command`, `open_terminal`, `get_output`, `send_input`
- **Git**: `git_status`, `git_add`, `git_commit`, `git_diff`, `git_branch`
- **Search**: `find_in_file`, `find_in_project`, `replace_text`, `regex_search`
- **Sync**: `get_cursor_pos`, `set_cursor_pos`, `get_selection`, `sync_viewport`
- **Advanced**: `get_code_lens`, `apply_quick_fix`, `refactor_symbol`, `manage_plugins`

### Results (plugin → agent)

- `result`:

```json
{
  "type":"result",
  "id":"uuid",
  "status":"ok" | "error",
  "data": {...} | {"message":"..."}
}
```

### Events (plugin → sub-agent)

Real-time state synchronization for sub-agents:

- `event`:

```json
{
  "type":"event",
  "event": "editor_state" | "file_system" | "project_state" |
           "terminal_output" | "git_state" | "code_analysis" | "user_interaction",
  "data": {...}
}
```

**Event Categories**:

- **Editor State**: `buffer_changed`, `cursor_moved`, `selection_changed`, `viewport_changed`, `file_saved`, `file_opened`, `file_closed`
- **File System**: `file_created`, `file_deleted`, `file_renamed`, `directory_changed`, `file_watched`
- **Project**: `build_started`, `build_completed`, `test_results`, `workspace_changed`, `config_updated`
- **Terminal**: `command_executed`, `output_received`, `process_started`, `process_ended`
- **Git**: `branch_changed`, `commit_made`, `status_updated`, `merge_conflict`, `push_completed`
- **Code Analysis**: `errors_detected`, `completions_available`, `symbols_updated`, `lint_results`
- **User Interaction**: `plugin_loaded`, `settings_changed`, `pane_split`, `tab_switched`

Bridge acts as proxy: validates `action` against `allowed_scopes` before forwarding to plugin.

---

## Plugin Protocol (bridge ↔ plugin)

- Plugin registers to bridge on start (WS client to `ws://HOST:PORT/plugin/{plugin_id}`).
- Same messages as above (`cmd`, `result`, `event`).
- Plugin must validate paths & reject paths outside `ALLOWED_ROOT`.

---

## Policy & Security Rules

1. **Auth & secrets**

   - Management endpoints protected by `MCP_TOKEN` (high privilege).
   - Sessions use ephemeral `session_token` (Bearer) with TTL.

2. **Scopes**

   - **Sub-Agent Scopes (Full IDE Capabilities)**:
     - File Operations: `read:files`, `write:files`, `create:files`, `delete:files`, `rename:files`, `search:files`
     - Buffer Management: `read:buffers`, `edit:buffers`, `manage:buffers`, `split:panes`, `manage:tabs`
     - Code Analysis: `analyze:syntax`, `detect:errors`, `suggest:completion`, `format:code`, `navigate:symbols`
     - Project Management: `explore:project`, `manage:workspace`, `watch:changes`, `build:project`
     - Terminal & Execution: `exec:terminal`, `run:commands`, `debug:session`, `test:code`
     - Version Control: `git:status`, `git:commit`, `git:branch`, `view:diffs`
     - Search & Replace: `search:content`, `replace:text`, `regex:operations`, `search:project`
     - Real-time Sync: `sync:cursor`, `sync:selection`, `sync:viewport`, `monitor:state`
     - Advanced Features: `code:lens`, `quick:fixes`, `refactor:code`, `manage:plugins`

3. **Roots**

   - Paths must be within `roots`. Implement canonicalization & avoid symlink attacks: use `os.path.realpath` & validate prefix.

4. **Limits**

   - `max_edit_bytes` per edit (e.g., 100KB).
   - `rate_limit_per_session` (e.g., 10 req/s).
   - `max_concurrent_sessions` configurable.

5. **Confirmation**

   - Destructive actions (delete file, run shell) must require explicit UI confirmation.

6. **Transport**

   - Use Unix domain sockets if possible between bridge & plugin.
   - Use TLS if traffic crosses machine boundary.

7. **Audit & Logging**
   - All actions logged to append-only JSONL with timestamps.
   - Provide `logs` endpoint for admin review (protected).

---

## Session lifecycle (state diagram — textual)

1. `REQUESTED` — agent posts request.
2. `PENDING_APPROVAL` — bridge notifies Cursor UI.
3. `APPROVED` — Cursor calls approve → bridge creates session.
4. `ACTIVE` — agent connects with token; bridge proxies messages to plugin.
5. `EXPIRED` or `REVOKED` — bridge closes WS, invalidates token, logs termination.

---

## Failure modes & mitigation

- **Agent attempts out-of-scope action** → bridge responds `error: forbidden`.
- **Plugin disconnects mid-session** → bridge marks session degraded; attempts reconnect; notifies agent & Cursor.
- **Token leak** → tokens have short TTL; session revoke available; logs show last actions.
- **Path traversal** → validate canonical path; reject if outside allowed root.

---

## Concurrency & consistency

- Per-buffer lock: plugin should implement lightweight locks to avoid race edits.
- If multi-agent editing required, consider CRDT/OT lib (advanced).

---

## Testing checklist (E2E) - Sub-Agent Capabilities

**Basic Flow**:

- [ ] Request access → PENDING visible to Cursor.
- [ ] Approve → Session token returned.
- [ ] Sub-agent WS connect with token → success.
- [ ] Revoke session → WS closed & actions blocked.

**File Operations**:

- [ ] Sub-agent open_file inside root → success.
- [ ] Sub-agent tries open_file outside root → 403 error.
- [ ] Sub-agent create/delete/rename files → works within scope.
- [ ] Sub-agent search_files → returns filtered results.

**Buffer Management**:

- [ ] Sub-agent get_buffers → returns current buffers.
- [ ] Sub-agent edit_buffer → changes reflected in VoidEditor.
- [ ] Sub-agent switch_buffer → cursor moves to correct buffer.
- [ ] Sub-agent split_pane → new pane created.

**Code Analysis**:

- [ ] Sub-agent analyze_syntax → returns syntax tree.
- [ ] Sub-agent get_errors → returns diagnostic info.
- [ ] Sub-agent get_completions → returns code suggestions.
- [ ] Sub-agent format_code → code formatted correctly.

**Project Management**:

- [ ] Sub-agent explore_tree → returns project structure.
- [ ] Sub-agent build_project → build executed successfully.
- [ ] Sub-agent run_tests → test results returned.

**Terminal Integration**:

- [ ] Sub-agent exec_command → command executed in VoidEditor terminal.
- [ ] Sub-agent get_output → terminal output received.

**Real-time Sync**:

- [ ] Cursor movement → sub-agent receives cursor_moved event.
- [ ] File changes → sub-agent receives buffer_changed event.
- [ ] Manual sync → sub-agent can query current state.

**Security & Scope**:

- [ ] Sub-agent exceeds scope limits → actions rejected.
- [ ] Edit size > max_edit_bytes → rejected.
- [ ] Rate limiting → excessive requests throttled.
- [ ] Audit log contains entries for each sub-agent action.

---

## Metrics & monitoring (optional)

- Track number of requests, approvals, session durations, tokens issued, failed auth attempts.
- Expose `/metrics` for Prometheus (lab only).

---

## Integration points with Cursor & VoidEditor

- Cursor:

  - UI: modal consent; show scopes & reason; approve/deny; show session timer & revoke.
  - Background: connect to SSE/WS for `PENDING` events.

- VoidEditor:
  - Best: plugin using native editor API to open buffers & apply edits (no file-reload hacks).
  - Fallback: use filesystem and trigger editor refresh, or CLI invocation.

---

## Example message flows - Sub-Agent Operations

**Initial Setup**:

1. Agent (Cursor) → `POST /mcp/request_access` with sub-agent scopes → Bridge stores request → emits event to Cursor SSE.
2. User (Cursor) → UI approve → `POST /mcp/approve` → Bridge creates session + token → Cursor shows session info.
3. Sub-Agent → WS connect `/mcp/session/{id}` with `Bearer token` → Bridge authenticates → Sub-agent runtime ready.

**Sub-Agent Development Flow**: 4. Sub-Agent → `cmd explore_tree` → Plugin returns project structure → Sub-agent understands codebase. 5. Sub-Agent → `cmd open_file main.py` → Plugin opens file → Sub-agent reads code. 6. Sub-Agent → `cmd analyze_syntax` → Plugin returns AST + errors → Sub-agent understands issues. 7. Sub-Agent → `cmd edit_buffer {refactored_code}` → Plugin applies changes → VoidEditor updates display. 8. Plugin → `event buffer_changed` → Sub-agent receives confirmation → continues workflow.

**Real-time Collaboration**: 9. User edits file manually → Plugin → `event cursor_moved` → Sub-agent adapts behavior. 10. Sub-Agent → `cmd build_project` → Plugin runs build → Sub-agent gets results → continues development.

**Advanced Operations**: 11. Sub-Agent → `cmd exec_command "npm test"` → Plugin runs tests → Sub-agent analyzes test results. 12. Sub-Agent → `cmd git_commit {message}` → Plugin commits changes → Sub-agent tracks version control.

---

## Extensions & future work

- Persistent DB for requests & audits (sqlite/postgres).
- Fine-grained role-based policies (RBAC).
- Multi-editor collaborative editing via CRDT.
- UI plugin for Cursor to visualize audit & replays.

---

## Appendices

### Sub-Agent Scope Presets (suggested)

**Basic Sub-Agent** (read-only):

- `read:files`, `read:buffers`, `search:files`, `analyze:syntax`, `sync:cursor`

**Standard Sub-Agent** (development):

- `read:files`, `write:files`, `edit:buffers`, `manage:buffers`, `format:code`
- `search:content`, `sync:cursor`, `sync:selection`, `explore:project`

**Advanced Sub-Agent** (full IDE):

- `create:files`, `delete:files`, `manage:workspace`, `build:project`, `run:tests`
- `exec:terminal`, `git:*`, `debug:session`, `refactor:code`, `manage:plugins`

**Restricted Sub-Agent** (sandboxed):

- `read:files`, `analyze:syntax`, `search:content` (no write/exec permissions)

### Sample audit log entry

```json
{
  "ts": "2025-09-21T09:00:00Z",
  "actor": "agent-1",
  "user": "minixo",
  "action": "open_file",
  "args": { "path": "/home/minixo/lab/projects/x/README.md" },
  "result": "ok",
  "session_id": "uuid",
  "request_id": "uuid"
}
```

---

This document is intended as a complete technical guide for prototyping MCP bridge.

**Implementation ready?** This blueprint provides complete specifications for:

- `bridge_app.py` (FastAPI) with all endpoints & WS proxy
- `plugin_void.js` (Node) stub implementing the plugin protocol above
- Example Cursor UI (HTML + JS) receiving SSE/WS notifications and displaying consent modal

See **[README.md](README.md)** for quick start guide or **[docs_id/README.md](docs_id/README.md)** for Indonesian version.
