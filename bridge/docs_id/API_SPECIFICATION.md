# MCP VoidEditor Bridge — Spesifikasi API

> **Bahasa**: [🇺🇸 English](../API_SPECIFICATION.md) | [🇮🇩 Bahasa Indonesia](API_SPECIFICATION.md)

**Dokumentasi API lengkap untuk sistem MCP Bridge yang memungkinkan sub-agent di VoidEditor**

## 📡 Gambaran API

Sistem MCP Bridge mengekspos beberapa interface API:

1. **Management API** - Untuk Cursor mengelola request akses dan sesi
2. **WebSocket API** - Untuk komunikasi real-time dengan agent dan plugin
3. **Events API** - Untuk notifikasi real-time via Server-Sent Events
4. **Internal Protocol** - Untuk komunikasi agent-plugin

---

## 🌐 Management API (Interface Cursor)

### Base URL

```
http://127.0.0.1:8787
```

### Autentikasi

- **Diperlukan**: Header `Authorization: Bearer ${MCP_TOKEN}` untuk semua endpoint management
- **Keamanan**: MCP_TOKEN harus dijaga kerahasiaannya dan tidak di-commit ke version control

---

## 🎯 Manajemen Request Akses

### `POST /mcp/request_access`

**Request akses ke VoidEditor untuk operasi sub-agent**

#### Format Request

```http
POST /mcp/request_access HTTP/1.1
Content-Type: application/json
Authorization: Bearer your-mcp-token

{
  "agent_id": "sub-agent-dev-1",
  "scopes": [
    "read:files",
    "edit:buffers",
    "exec:terminal",
    "git:operations"
  ],
  "root_path": "/workspace/project",
  "ttl": 300,
  "reason": "Development workflow automation"
}
```

#### Parameter Request

| Field       | Type    | Required | Description                             |
| ----------- | ------- | -------- | --------------------------------------- |
| `agent_id`  | string  | ✅       | Identifier unik untuk agent             |
| `scopes`    | array   | ✅       | Daftar izin yang diminta                |
| `root_path` | string  | ✅       | Path root yang diizinkan                |
| `ttl`       | integer | ❌       | Time-to-live dalam detik (default: 300) |
| `reason`    | string  | ❌       | Alasan request akses                    |

#### Response Success

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "request_id": "req_123456789",
  "status": "pending",
  "message": "Access request submitted for user approval"
}
```

#### Response Error

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_request",
  "message": "Invalid scopes or root path"
}
```

### `POST /mcp/approve`

**Menyetujui request akses**

#### Request Format

```http
POST /mcp/approve HTTP/1.1
Content-Type: application/json
Authorization: Bearer your-mcp-token

{
  "request_id": "req_123456789",
  "approved_scopes": [
    "read:files",
    "edit:buffers"
  ]
}
```

#### Response Success

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "session_id": "sess_987654321",
  "token": "token_abc123",
  "expires_at": "2024-01-15T10:30:00Z",
  "scopes": ["read:files", "edit:buffers"]
}
```

### `POST /mcp/deny`

**Menolak request akses**

#### Request Format

```http
POST /mcp/deny HTTP/1.1
Content-Type: application/json
Authorization: Bearer your-mcp-token

{
  "request_id": "req_123456789",
  "reason": "Security policy violation"
}
```

---

## 🔌 WebSocket API (Real-time Communication)

### Connection URL

```
ws://127.0.0.1:8787/ws
```

### Authentication

```json
{
  "type": "auth",
  "token": "your-session-token"
}
```

### Message Types

#### Command Message (Agent → Plugin)

```json
{
  "type": "cmd",
  "session_id": "sess_987654321",
  "action": "file_operation",
  "params": {
    "operation": "read",
    "path": "/workspace/file.js"
  },
  "timestamp": "2024-01-15T10:00:00Z"
}
```

#### Result Message (Plugin → Agent)

```json
{
  "type": "result",
  "session_id": "sess_987654321",
  "request_id": "cmd_123",
  "status": "success",
  "data": {
    "content": "console.log('Hello World');",
    "size": 25
  },
  "timestamp": "2024-01-15T10:00:01Z"
}
```

#### Event Message (Plugin → Agent)

```json
{
  "type": "event",
  "session_id": "sess_987654321",
  "event_type": "editor_state",
  "data": {
    "cursor_position": { "line": 10, "column": 5 },
    "selection": { "start": 100, "end": 150 }
  },
  "timestamp": "2024-01-15T10:00:02Z"
}
```

---

## 📡 Events API (Server-Sent Events)

### Connection URL

```
http://127.0.0.1:8787/events
```

### Event Types

#### Access Request Event

```http
data: {"type": "access_request", "request_id": "req_123456789", "agent_id": "sub-agent-dev-1", "scopes": ["read:files"], "timestamp": "2024-01-15T10:00:00Z"}

```

#### Session Created Event

```http
data: {"type": "session_created", "session_id": "sess_987654321", "agent_id": "sub-agent-dev-1", "expires_at": "2024-01-15T10:30:00Z"}

```

#### Session Expired Event

```http
data: {"type": "session_expired", "session_id": "sess_987654321", "reason": "ttl_reached"}

```

---

## 🎮 Command Actions

### File Operations

#### Read File

```json
{
  "action": "file_operation",
  "params": {
    "operation": "read",
    "path": "/workspace/file.js"
  }
}
```

#### Write File

```json
{
  "action": "file_operation",
  "params": {
    "operation": "write",
    "path": "/workspace/file.js",
    "content": "console.log('Hello World');"
  }
}
```

#### List Directory

```json
{
  "action": "file_operation",
  "params": {
    "operation": "list",
    "path": "/workspace"
  }
}
```

### Buffer Management

#### Get Buffer Content

```json
{
  "action": "buffer_management",
  "params": {
    "operation": "get_content",
    "buffer_id": "buf_123"
  }
}
```

#### Update Buffer

```json
{
  "action": "buffer_management",
  "params": {
    "operation": "update",
    "buffer_id": "buf_123",
    "content": "new content"
  }
}
```

### Terminal Operations

#### Execute Command

```json
{
  "action": "terminal_operation",
  "params": {
    "operation": "execute",
    "command": "npm install",
    "cwd": "/workspace"
  }
}
```

### Git Operations

#### Git Status

```json
{
  "action": "git_operation",
  "params": {
    "operation": "status",
    "path": "/workspace"
  }
}
```

#### Git Commit

```json
{
  "action": "git_operation",
  "params": {
    "operation": "commit",
    "message": "Add new feature",
    "path": "/workspace"
  }
}
```

---

## 🔍 Scope Definitions

### File Operations

- `read:files` - Membaca file dan direktori
- `write:files` - Menulis dan membuat file
- `delete:files` - Menghapus file dan direktori

### Buffer Management

- `read:buffers` - Membaca konten buffer
- `edit:buffers` - Mengedit konten buffer
- `create:buffers` - Membuat buffer baru

### Terminal Operations

- `exec:terminal` - Menjalankan perintah terminal
- `read:terminal` - Membaca output terminal

### Git Operations

- `git:read` - Membaca status Git
- `git:write` - Melakukan commit dan push

### Project Operations

- `read:project` - Membaca struktur proyek
- `build:project` - Build proyek
- `test:project` - Menjalankan test

---

## ⚠️ Error Handling

### Error Response Format

```json
{
  "error": "error_code",
  "message": "Human readable error message",
  "details": {
    "field": "additional error details"
  },
  "timestamp": "2024-01-15T10:00:00Z"
}
```

### Common Error Codes

| Code              | HTTP Status | Description                       |
| ----------------- | ----------- | --------------------------------- |
| `invalid_request` | 400         | Request format tidak valid        |
| `unauthorized`    | 401         | Token tidak valid atau expired    |
| `forbidden`       | 403         | Tidak memiliki izin untuk operasi |
| `not_found`       | 404         | Resource tidak ditemukan          |
| `session_expired` | 410         | Sesi telah expired                |
| `rate_limited`    | 429         | Terlalu banyak request            |
| `internal_error`  | 500         | Error internal server             |

---

## 🔒 Security Considerations

### Token Security

- Token harus di-generate secara acak
- Token memiliki TTL yang terbatas
- Token harus di-revoke setelah digunakan

### Scope Validation

- Setiap operasi divalidasi terhadap scope yang disetujui
- Root path restriction untuk mencegah akses ke luar workspace
- Audit logging untuk semua operasi

### Rate Limiting

- Maksimal 100 request per menit per session
- Burst allowance untuk operasi batch
- Graceful degradation saat rate limit tercapai

---

## 📊 Performance Specifications

### Response Times

- Management API: < 100ms
- WebSocket messages: < 50ms
- File operations: < 200ms
- Terminal operations: < 500ms

### Throughput

- Concurrent sessions: 10
- Messages per second: 100
- File operations per minute: 1000

---

**API ini dirancang untuk memberikan kontrol sub-agent yang aman dan efisien di VoidEditor dengan keamanan dan auditabilitas yang ketat.**
