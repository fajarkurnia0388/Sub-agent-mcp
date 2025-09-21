# MCP VoidEditor Bridge — API Specification

> **Language**: [🇺🇸 English](API_SPECIFICATION.md) | [🇮🇩 Bahasa Indonesia](docs_id/API_SPECIFICATION.md)

**Complete API documentation for the MCP Bridge system enabling sub-agents in VoidEditor**

## 📡 API Overview

The MCP Bridge system exposes multiple API interfaces:

1. **Management API** - For Cursor to manage access requests and sessions
2. **WebSocket API** - For real-time communication with agents and plugins
3. **Events API** - For real-time notifications via Server-Sent Events
4. **Internal Protocol** - For agent-plugin communication

---

## 🌐 Management API (Cursor Interface)

### Base URL

```
http://127.0.0.1:8787
```

### Authentication

- **Required**: `Authorization: Bearer ${MCP_TOKEN}` header for all management endpoints
- **Security**: MCP_TOKEN must be kept secret and not committed to version control

---

## 🎯 Access Request Management

### `POST /mcp/request_access`

**Request access to VoidEditor for sub-agent operations**

#### Request Format

```http
POST /mcp/request_access HTTP/1.1
Content-Type: application/json
Authorization: Bearer your-mcp-token

{
  "agent_id": "sub-agent-dev-1",
  "scopes": [
    "read:files",
    "edit:buffers",
    "analyze:syntax",
    "build:project",
    "git:commit"
  ],
  "roots": [
    "/home/user/projects/myrepo"
  ],
  "reason": "Sub-agent development workflow in VoidEditor"
}
```

#### Request Parameters

| Parameter  | Type   | Required | Description                                |
| ---------- | ------ | -------- | ------------------------------------------ |
| `agent_id` | string | Yes      | Unique identifier for the requesting agent |
| `scopes`   | array  | Yes      | List of requested permissions              |
| `roots`    | array  | Yes      | Allowed file system paths                  |
| `reason`   | string | Yes      | Human-readable justification for access    |

#### Available Scopes

**File Operations**:

- `read:files` - Read file contents
- `write:files` - Write file contents
- `create:files` - Create new files
- `delete:files` - Delete files
- `rename:files` - Rename/move files
- `search:files` - Search within files

**Buffer Management**:

- `read:buffers` - Read editor buffers
- `edit:buffers` - Edit buffer contents
- `manage:buffers` - Create/close/switch buffers
- `split:panes` - Split editor panes
- `manage:tabs` - Manage editor tabs

**Code Analysis**:

- `analyze:syntax` - Perform syntax analysis
- `detect:errors` - Detect code errors
- `suggest:completion` - Provide code completions
- `format:code` - Format code
- `navigate:symbols` - Navigate code symbols

**Project Management**:

- `explore:project` - Explore project structure
- `manage:workspace` - Manage workspace settings
- `watch:changes` - Watch file changes
- `build:project` - Build/compile project

**Terminal & Execution**:

- `exec:terminal` - Execute terminal commands
- `run:commands` - Run specific commands
- `debug:session` - Debug programs
- `test:code` - Run tests

**Version Control**:

- `git:status` - Get git status
- `git:commit` - Make git commits
- `git:branch` - Manage git branches
- `view:diffs` - View code diffs

#### Success Response

```json
{
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "status": "pending",
  "created_at": "2024-01-01T12:00:00Z"
}
```

#### Error Responses

**400 Bad Request**

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Invalid scopes provided",
    "details": {
      "invalid_scopes": ["invalid:scope"]
    }
  }
}
```

**401 Unauthorized**

```json
{
  "error": {
    "code": "unauthorized",
    "message": "Invalid or missing MCP token"
  }
}
```

### `GET /mcp/requests`

**List recent access requests**

#### Query Parameters

| Parameter | Type    | Default | Description                                       |
| --------- | ------- | ------- | ------------------------------------------------- |
| `status`  | string  | all     | Filter by status: `pending`, `approved`, `denied` |
| `limit`   | integer | 50      | Maximum number of requests to return              |
| `offset`  | integer | 0       | Number of requests to skip                        |

#### Response Format

```json
{
  "requests": [
    {
      "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
      "agent_id": "sub-agent-dev-1",
      "scopes": ["read:files", "edit:buffers"],
      "roots": ["/home/user/projects/myrepo"],
      "reason": "Sub-agent development workflow",
      "status": "pending",
      "created_at": "2024-01-01T12:00:00Z",
      "approved_by": null,
      "session_id": null
    }
  ],
  "total": 1,
  "has_more": false
}
```

### `POST /mcp/approve`

**Approve an access request and create session**

#### Request Format

```json
{
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "approved_scopes": ["read:files", "edit:buffers", "analyze:syntax"],
  "ttl_seconds": 300
}
```

#### Request Parameters

| Parameter         | Type    | Required | Description                                             |
| ----------------- | ------- | -------- | ------------------------------------------------------- |
| `request_id`      | string  | Yes      | ID of the request to approve                            |
| `approved_scopes` | array   | No       | Subset of requested scopes to approve (defaults to all) |
| `ttl_seconds`     | integer | No       | Session duration in seconds (default: 300)              |

#### Success Response

```json
{
  "session_id": "sess-123e4567-e89b-12d3-a456-426614174000",
  "session_token": "tok-abcd1234-5678-90ef-ghij-klmnopqrstuv",
  "expires_at": "2024-01-01T12:05:00Z",
  "approved_scopes": ["read:files", "edit:buffers", "analyze:syntax"]
}
```

### `POST /mcp/deny`

**Deny an access request**

#### Request Format

```json
{
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "reason": "Insufficient justification"
}
```

#### Success Response

```json
{
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "status": "denied",
  "denied_at": "2024-01-01T12:00:00Z"
}
```

### `POST /mcp/revoke`

**Revoke an active session**

#### Request Format

```json
{
  "session_id": "sess-123e4567-e89b-12d3-a456-426614174000",
  "reason": "User requested termination"
}
```

#### Success Response

```json
{
  "session_id": "sess-123e4567-e89b-12d3-a456-426614174000",
  "status": "revoked",
  "revoked_at": "2024-01-01T12:00:00Z"
}
```

### `GET /mcp/sessions`

**List active sessions**

#### Response Format

```json
{
  "sessions": [
    {
      "session_id": "sess-123e4567-e89b-12d3-a456-426614174000",
      "agent_id": "sub-agent-dev-1",
      "status": "active",
      "created_at": "2024-01-01T12:00:00Z",
      "expires_at": "2024-01-01T12:05:00Z",
      "last_activity": "2024-01-01T12:02:30Z",
      "approved_scopes": ["read:files", "edit:buffers"],
      "allowed_roots": ["/home/user/projects/myrepo"],
      "request_count": 15
    }
  ],
  "total": 1
}
```

---

## 📡 Events API (Real-time Notifications)

### `GET /mcp/events`

**Server-Sent Events stream for real-time notifications**

#### Connection

```http
GET /mcp/events HTTP/1.1
Authorization: Bearer your-mcp-token
Accept: text/event-stream
```

#### Event Types

**New Request Event**

```
event: request_created
data: {
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "agent_id": "sub-agent-dev-1",
  "scopes": ["read:files", "edit:buffers"],
  "reason": "Development workflow",
  "created_at": "2024-01-01T12:00:00Z"
}
```

**Request Status Changed**

```
event: request_status_changed
data: {
  "request_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "old_status": "pending",
  "new_status": "approved",
  "changed_at": "2024-01-01T12:01:00Z"
}
```

**Session Created**

```
event: session_created
data: {
  "session_id": "sess-123e4567-e89b-12d3-a456-426614174000",
  "agent_id": "sub-agent-dev-1",
  "expires_at": "2024-01-01T12:05:00Z"
}
```

**Session Expired/Revoked**

```
event: session_ended
data: {
  "session_id": "sess-123e4567-e89b-12d3-a456-426614174000",
  "reason": "expired",
  "ended_at": "2024-01-01T12:05:00Z"
}
```

### `WS /mcp/events/ws`

**WebSocket alternative for real-time notifications**

Same event format as SSE, but delivered via WebSocket messages.

---

## 🔌 WebSocket API (Agent Communication)

### Agent Session WebSocket

#### Connection

```
ws://127.0.0.1:8787/mcp/session/{session_id}
```

#### Headers

```
Authorization: Bearer {session_token}
```

#### Message Format

All messages are JSON with the following structure:

```typescript
interface Message {
  type: "cmd" | "result" | "event";
  id: string; // UUID for request/response matching
  [key: string]: any; // Message-specific fields
}
```

### Commands (Agent → Bridge → Plugin)

#### File Operations

**Open File**

```json
{
  "type": "cmd",
  "id": "cmd-001",
  "action": "open_file",
  "args": {
    "path": "src/main.py",
    "mode": "read"
  }
}
```

**Create File**

```json
{
  "type": "cmd",
  "id": "cmd-002",
  "action": "create_file",
  "args": {
    "path": "src/new_module.py",
    "content": "# New Python module\n"
  }
}
```

**Delete File**

```json
{
  "type": "cmd",
  "id": "cmd-003",
  "action": "delete_file",
  "args": {
    "path": "temp/old_file.txt"
  }
}
```

#### Buffer Management

**Edit Buffer**

```json
{
  "type": "cmd",
  "id": "cmd-004",
  "action": "edit_buffer",
  "args": {
    "buffer_id": "main.py",
    "changes": [
      {
        "start_line": 10,
        "end_line": 12,
        "new_text": "def improved_function():\n    return 'better implementation'"
      }
    ]
  }
}
```

**Get Buffers**

```json
{
  "type": "cmd",
  "id": "cmd-005",
  "action": "get_buffers",
  "args": {}
}
```

#### Code Analysis

**Analyze Syntax**

```json
{
  "type": "cmd",
  "id": "cmd-006",
  "action": "analyze_syntax",
  "args": {
    "file": "src/main.py",
    "include_ast": true
  }
}
```

**Get Completions**

```json
{
  "type": "cmd",
  "id": "cmd-007",
  "action": "get_completions",
  "args": {
    "file": "src/main.py",
    "line": 25,
    "column": 10
  }
}
```

#### Project Operations

**Explore Tree**

```json
{
  "type": "cmd",
  "id": "cmd-008",
  "action": "explore_tree",
  "args": {
    "path": "src/",
    "max_depth": 3,
    "include_hidden": false
  }
}
```

**Build Project**

```json
{
  "type": "cmd",
  "id": "cmd-009",
  "action": "build_project",
  "args": {
    "target": "development",
    "clean": false
  }
}
```

#### Terminal Operations

**Execute Command**

```json
{
  "type": "cmd",
  "id": "cmd-010",
  "action": "exec_command",
  "args": {
    "command": "python -m pytest tests/",
    "cwd": "/home/user/projects/myrepo",
    "timeout": 30
  }
}
```

### Results (Plugin → Bridge → Agent)

#### Success Result

```json
{
  "type": "result",
  "id": "cmd-001",
  "status": "ok",
  "data": {
    "content": "# Main Python file\nimport sys\n...",
    "size": 1024,
    "last_modified": "2024-01-01T11:30:00Z"
  }
}
```

#### Error Result

```json
{
  "type": "result",
  "id": "cmd-002",
  "status": "error",
  "error": {
    "code": "file_not_found",
    "message": "File does not exist: nonexistent.py",
    "details": {
      "path": "nonexistent.py",
      "checked_locations": ["/home/user/projects/myrepo/nonexistent.py"]
    }
  }
}
```

### Events (Plugin → Bridge → Agent)

#### Editor State Events

**Buffer Changed**

```json
{
  "type": "event",
  "event": "buffer_changed",
  "data": {
    "buffer_id": "main.py",
    "changes": [
      {
        "start_line": 15,
        "end_line": 15,
        "new_text": "    # Added comment\n"
      }
    ],
    "timestamp": "2024-01-01T12:03:00Z"
  }
}
```

**Cursor Moved**

```json
{
  "type": "event",
  "event": "cursor_moved",
  "data": {
    "buffer_id": "main.py",
    "line": 20,
    "column": 15,
    "timestamp": "2024-01-01T12:03:01Z"
  }
}
```

#### Project Events

**Build Completed**

```json
{
  "type": "event",
  "event": "build_completed",
  "data": {
    "success": true,
    "duration_ms": 1500,
    "output": "Build successful\n",
    "warnings": 0,
    "errors": 0
  }
}
```

**Test Results**

```json
{
  "type": "event",
  "event": "test_results",
  "data": {
    "passed": 25,
    "failed": 2,
    "skipped": 1,
    "duration_ms": 3000,
    "failures": [
      {
        "test": "test_authentication",
        "error": "AssertionError: Expected 200, got 401"
      }
    ]
  }
}
```

---

## 🔌 Plugin WebSocket API

### Plugin Registration

#### Connection

```
ws://127.0.0.1:8787/plugin/{plugin_id}
```

#### Registration Message

```json
{
  "type": "register",
  "plugin_id": "voideditor-plugin-1",
  "capabilities": [
    "file_operations",
    "buffer_management",
    "code_analysis",
    "terminal_execution"
  ],
  "version": "1.0.0"
}
```

#### Registration Response

```json
{
  "type": "registered",
  "status": "ok",
  "assigned_id": "voideditor-plugin-1",
  "bridge_version": "1.0.0"
}
```

### Command Handling

Plugins receive the same command format as shown in the Agent API section and must respond with appropriate results.

---

## 📊 Rate Limiting

### Rate Limit Headers

```http
X-RateLimit-Limit: 10            # Requests per window
X-RateLimit-Remaining: 7         # Remaining requests
X-RateLimit-Reset: 1704110460    # Reset timestamp
X-RateLimit-Window: 60           # Window duration (seconds)
```

### Rate Limit Response

```json
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Too many requests",
    "retry_after": 30,
    "limit": 10,
    "window_seconds": 60
  }
}
```

---

## 🔒 Security Headers

### Request Headers

```http
Authorization: Bearer your-mcp-token    # Required for management endpoints
X-Forwarded-For: 127.0.0.1             # Must be localhost
User-Agent: Cursor/1.0                  # Client identification
```

### Response Headers

```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Access-Control-Allow-Origin: http://127.0.0.1:*
Content-Security-Policy: default-src 'self'
```

---

## ⚡ Performance Specifications

### Timeouts

- **HTTP Request timeout**: 30 seconds
- **WebSocket ping timeout**: 10 seconds
- **Session cleanup interval**: 60 seconds
- **Event delivery timeout**: 5 seconds

### Limits

- **Maximum concurrent sessions**: 10 (configurable)
- **Maximum request size**: 10MB
- **Maximum message size**: 1MB
- **Maximum file edit size**: 100KB
- **Rate limit**: 10 requests per 60 seconds per session

### Response Times (SLA)

- **Management API**: < 500ms
- **Session creation**: < 1s
- **Command forwarding**: < 100ms
- **Event delivery**: < 50ms

---

## 🐛 Error Codes

### HTTP Error Codes

| Code | Description                                           |
| ---- | ----------------------------------------------------- |
| 400  | Bad Request - Invalid request format                  |
| 401  | Unauthorized - Invalid or missing token               |
| 403  | Forbidden - Insufficient permissions                  |
| 404  | Not Found - Resource does not exist                   |
| 409  | Conflict - Resource already exists                    |
| 429  | Too Many Requests - Rate limit exceeded               |
| 500  | Internal Server Error - Server error                  |
| 503  | Service Unavailable - Service temporarily unavailable |

### WebSocket Close Codes

| Code | Description                              |
| ---- | ---------------------------------------- |
| 1000 | Normal Closure                           |
| 1001 | Going Away                               |
| 1008 | Policy Violation - Invalid session/token |
| 1011 | Server Error                             |
| 4000 | Session Expired                          |
| 4001 | Session Revoked                          |
| 4002 | Invalid Scope                            |

---

**Next**: See [DEPLOYMENT.md](DEPLOYMENT.md) for deployment and security guidelines
