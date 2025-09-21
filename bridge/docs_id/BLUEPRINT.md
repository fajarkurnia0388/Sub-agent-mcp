# MCP Bridge — BLUEPRINT (Technical Design)

> **Bahasa**: [🇺🇸 English](../BLUEPRINT.md) | [🇮🇩 Bahasa Indonesia](BLUEPRINT.md)

Dokumen ini adalah blueprint teknis untuk **MCP Bridge** yang memungkinkan **sub-agen** beroperasi di dalam VoidEditor dengan kemampuan IDE lengkap, seperti agen bekerja di Cursor. Fokus: arsitektur, API spec, pesan WS, lifecycle, keamanan, testing, dan edge-cases.

---

## Tujuan

Sediakan mediator aman dan auditable yang:

- menerima request akses dari agen,
- meminta persetujuan user lewat Cursor,
- menciptakan session ephemeral untuk **sub-agen** beroperasi di dalam VoidEditor,
- meneruskan perintah dengan enforcement policy (roots/scopes/limits).

---

## Komponen & Tanggung Jawab

1. **Cursor (MCP client & UI)**

   - Kirim `request_access`.
   - Tampilkan modal approval.
   - Panggil `approve/deny`.
   - Tampilkan session status & revoke control.

2. **Bridge (FastAPI)**

   - Endpoint manajemen: request, list, approve, deny, revoke.
   - `events` stream ke Cursor (SSE atau WS).
   - WebSocket server untuk session agent (`/mcp/session/{session_id}`).
   - WebSocket client/server untuk plugin (`/plugin/{plugin_id}`) atau connector ke plugin socket.
   - Policy enforcement, audit logs, session lifecycle.

3. **VoidEditor Plugin (extension)**
   - Terhubung ke bridge (persistent WS).
   - Menangani aksi: open_file, edit_buffer, get_buffers, cursor_move.
   - Emit events: buffer_changed, file_saved, error.

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

### Commands (agent → bridge/plugin)

- `cmd`:

```json
{
  "type":"cmd",
  "id":"uuid",
  "action":"open_file" | "edit_buffer" | "get_buffers" | "cursor_move",
  "args": { ... }
}
```

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

### Events (plugin → agent)

- `event`:

```json
{
  "type":"event",
  "event":"buffer_changed" | "file_saved" | "cursor_moved",
  "data": {...}
}
```

Bridge bertindak sebagai proxy: validasi `action` terhadap `allowed_scopes` sebelum forward ke plugin.

---

## Plugin Protocol (bridge ↔ plugin)

- Plugin mendaftar ke bridge saat start (WS client ke `ws://HOST:PORT/plugin/{plugin_id}`).
- Pesan yang sama seperti di atas (`cmd`, `result`, `event`).
- Plugin harus memvalidasi path & menolak path di luar `ALLOWED_ROOT`.

---

## Policy & Security Rules

1. **Auth & secrets**

   - Management endpoints protected by `MCP_TOKEN` (high privilege).
   - Sessions use ephemeral `session_token` (Bearer) with TTL.

2. **Scopes**

   - Granular scopes: `read:buffers`, `open:file`, `edit:buffer`, `list:files`, `exec:command` (last one disabled by default).

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

## Testing checklist (E2E)

- [ ] Request access → PENDING visible to Cursor.
- [ ] Approve → Session token returned.
- [ ] Agent WS connect with token → success.
- [ ] Agent open_file inside root → plugin opens & returns ok.
- [ ] Agent tries open_file outside root → 403 error.
- [ ] Agent edit_buffer > max_edit_bytes → rejected.
- [ ] Revoke session → WS closed & actions blocked.
- [ ] Plugin disconnect & reconnect → session handling validated.
- [ ] Audit log contains entries for each action.

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

## Example message flows (concise)

1. Agent → `POST /mcp/request_access` → Bridge stores request → emits event to Cursor SSE.
2. User (Cursor) → UI approve → `POST /mcp/approve` → Bridge create session + token → Cursor shows session info.
3. Agent → WS connect `/mcp/session/{id}` with `Bearer token` → Bridge authenticates → agent sends `cmd open_file` → bridge validates scope & path → forward to plugin WS → plugin opens file → sends `result ok` → bridge forwards to agent.

---

## Extensions & future work

- Persistent DB for requests & audits (sqlite/postgres).
- Fine-grained role-based policies (RBAC).
- Multi-editor collaborative editing via CRDT.
- UI plugin for Cursor to visualize audit & replays.

---

## Appendices

### Common scopes (suggested)

- `read:buffers`
- `open:file`
- `edit:buffer`
- `list:files`
- `exec:command` (dangerous; disabled by default)

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

Dokumen ini dimaksudkan menjadi panduan teknis lengkap untuk prototyping bridge MCP.

**Siap implementasi?** Blueprint ini menyediakan spesifikasi lengkap untuk:

- `bridge_app.py` (FastAPI) dengan semua endpoints & WS proxy
- `plugin_void.js` (Node) stub yang mengimplementasikan protokol plugin di atas
- Contoh Cursor UI (HTML + JS) yang menerima SSE/WS notifikasi dan menampilkan modal consent

Lihat **[README.md](README.md)** untuk panduan quick start atau **[../README.md](../README.md)** untuk versi bahasa Inggris.
