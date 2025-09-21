# MCP Local Provider — API Specification

> **Complete API documentation for the Local Provider system**

## 📡 API Overview

The Local Provider system exposes multiple API interfaces:

1. **External API** - For VoidEditor to interact with the "local model"
2. **Control API** - For MCP Bridge to manage sessions
3. **Internal MCP Protocol** - For Agent communication

---

## 🌐 External API (VoidEditor Interface)

### Base URL

```
http://127.0.0.1:11434
```

### Authentication

- **None required** for VoidEditor requests
- Session validation handled internally via registered sessions

---

## 🎯 Chat Completions API

### `POST /v1/chat/completions`

**OpenAI-compatible chat completions endpoint**

#### Request Format

```http
POST /v1/chat/completions HTTP/1.1
Content-Type: application/json
X-MCP-Session: optional-session-id

{
  "model": "cursor-agent",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful coding assistant."
    },
    {
      "role": "user",
      "content": "Explain async/await in Python"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "stream": false,
  "top_p": 0.9,
  "frequency_penalty": 0.0,
  "presence_penalty": 0.0
}
```

#### Request Parameters

| Parameter           | Type    | Required | Default        | Description                     |
| ------------------- | ------- | -------- | -------------- | ------------------------------- |
| `model`             | string  | No       | "cursor-agent" | Model identifier                |
| `messages`          | array   | Yes      | -              | Conversation messages           |
| `temperature`       | number  | No       | 0.8            | Sampling temperature (0.0-2.0)  |
| `max_tokens`        | integer | No       | 512            | Maximum response tokens         |
| `stream`            | boolean | No       | false          | Enable streaming response       |
| `top_p`             | number  | No       | 1.0            | Nucleus sampling parameter      |
| `frequency_penalty` | number  | No       | 0.0            | Frequency penalty (-2.0 to 2.0) |
| `presence_penalty`  | number  | No       | 0.0            | Presence penalty (-2.0 to 2.0)  |

#### Message Format

```typescript
interface Message {
  role: "system" | "user" | "assistant";
  content: string;
  name?: string; // Optional message author name
}
```

#### Success Response (Non-streaming)

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1677858242,
  "model": "cursor-agent",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "async/await is a syntax in Python for handling asynchronous operations..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 150,
    "total_tokens": 170
  }
}
```

#### Success Response (Streaming)

```
Content-Type: text/event-stream

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1677858242,"model":"cursor-agent","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1677858242,"model":"cursor-agent","choices":[{"index":0,"delta":{"content":"async"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1677858242,"model":"cursor-agent","choices":[{"index":0,"delta":{"content":"/await"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc123","object":"chat.completion.chunk","created":1677858242,"model":"cursor-agent","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

#### Error Responses

**503 Service Unavailable - No Active Session**

```json
{
  "error": {
    "message": "No agent session available",
    "type": "service_unavailable",
    "code": "no_active_session"
  }
}
```

**401 Unauthorized - Invalid Session**

```json
{
  "error": {
    "message": "Session token invalid or expired",
    "type": "invalid_session",
    "code": "session_expired"
  }
}
```

**429 Too Many Requests - Rate Limit**

```json
{
  "error": {
    "message": "Rate limit exceeded for session",
    "type": "rate_limit_exceeded",
    "code": "too_many_requests",
    "retry_after": 30
  }
}
```

**400 Bad Request - Invalid Input**

```json
{
  "error": {
    "message": "Invalid request format",
    "type": "invalid_request",
    "code": "bad_request",
    "details": "messages array is required"
  }
}
```

---

## 🦙 Ollama Compatibility API

### `POST /api/generate`

**Ollama-compatible generation endpoint (fallback)**

#### Request Format

```json
{
  "model": "cursor-agent",
  "prompt": "Explain async/await in Python",
  "stream": false,
  "options": {
    "temperature": 0.7,
    "max_tokens": 1024
  }
}
```

#### Response Format

```json
{
  "model": "cursor-agent",
  "created_at": "2024-01-01T12:00:00Z",
  "response": "async/await is a syntax in Python...",
  "done": true,
  "context": [1, 2, 3],
  "total_duration": 1500000000,
  "load_duration": 100000000,
  "prompt_eval_count": 20,
  "eval_count": 150
}
```

### `POST /api/chat`

**Ollama-compatible chat endpoint**

#### Request Format

```json
{
  "model": "cursor-agent",
  "messages": [
    {
      "role": "user",
      "content": "Hello, how are you?"
    }
  ],
  "stream": false
}
```

---

## 📋 Model Management API

### `GET /v1/models`

**List available models**

#### Response Format

```json
{
  "object": "list",
  "data": [
    {
      "id": "cursor-agent",
      "object": "model",
      "created": 1677610602,
      "owned_by": "mcp-local-provider",
      "permission": [],
      "root": "cursor-agent",
      "parent": null
    }
  ]
}
```

### `GET /api/tags`

**Ollama-compatible model list**

#### Response Format

```json
{
  "models": [
    {
      "name": "cursor-agent:latest",
      "modified_at": "2024-01-01T12:00:00Z",
      "size": 0,
      "digest": "sha256:abc123...",
      "details": {
        "format": "mcp-proxy",
        "family": "cursor",
        "families": ["cursor"],
        "parameter_size": "unknown",
        "quantization_level": "none"
      }
    }
  ]
}
```

---

## 🏥 Health & Status API

### `GET /health`

**System health check**

#### Response Format

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T12:00:00Z",
  "details": {
    "adapter": "operational",
    "mcp_bridge": "connected",
    "active_sessions": 2,
    "uptime_seconds": 3600
  }
}
```

### `GET /health/deep`

**Detailed health information**

#### Response Format

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T12:00:00Z",
  "components": {
    "adapter": {
      "status": "healthy",
      "active_connections": 5,
      "total_requests": 1250,
      "error_rate": 0.02
    },
    "mcp_bridge": {
      "status": "healthy",
      "connection_state": "connected",
      "last_ping": "2024-01-01T12:00:00Z",
      "latency_ms": 15
    },
    "sessions": {
      "active_count": 2,
      "expired_count": 5,
      "total_registered": 7
    }
  },
  "metrics": {
    "requests_per_second": 5.2,
    "average_latency_ms": 250,
    "memory_usage_mb": 128,
    "cpu_usage_percent": 15.5
  }
}
```

---

## 🔧 Control API (Internal)

### Base Authentication

```http
Authorization: Bearer {MCP_BRIDGE_TOKEN}
```

### `POST /control/register`

**Register agent session**

#### Request Format

```json
{
  "session_id": "sess-abc123",
  "session_token": "token-xyz789",
  "agent_id": "agent-cursor-1",
  "allowed_scopes": ["inference"],
  "ttl_seconds": 300,
  "label": "Cursor Agent - Code Assistant"
}
```

#### Response Format

```json
{
  "registered": "sess-abc123",
  "status": "active",
  "expires_at": "2024-01-01T12:05:00Z"
}
```

### `POST /control/revoke`

**Revoke agent session**

#### Request Format

```json
{
  "session_id": "sess-abc123",
  "reason": "user_request"
}
```

#### Response Format

```json
{
  "revoked": "sess-abc123",
  "status": "success"
}
```

### `GET /control/sessions`

**List active sessions**

#### Response Format

```json
{
  "sessions": [
    {
      "session_id": "sess-abc123",
      "agent_id": "agent-cursor-1",
      "status": "active",
      "created_at": "2024-01-01T12:00:00Z",
      "expires_at": "2024-01-01T12:05:00Z",
      "last_activity": "2024-01-01T12:02:30Z",
      "request_count": 15,
      "label": "Cursor Agent - Code Assistant"
    }
  ],
  "total_count": 1,
  "active_count": 1
}
```

---

## 🔗 Internal MCP Protocol

### Agent → Adapter Messages

#### Model Invoke Request

```json
{
  "type": "model_invoke",
  "id": "req-uuid-1234",
  "session_id": "sess-abc123",
  "model_meta": {
    "provider": "mcp-proxy",
    "label": "cursor-agent-as-model",
    "requested_scopes": ["inference"]
  },
  "payload": {
    "kind": "chat",
    "messages": [
      {
        "role": "user",
        "content": "Hello world"
      }
    ],
    "parameters": {
      "max_tokens": 512,
      "temperature": 0.7,
      "stream": false,
      "top_p": 0.9
    }
  }
}
```

### Adapter → Agent Messages

#### Model Result Response

```json
{
  "type": "model_result",
  "id": "req-uuid-1234",
  "status": "ok",
  "result": {
    "choices": [
      {
        "index": 0,
        "message": {
          "role": "assistant",
          "content": "Hello! How can I help you today?"
        },
        "finish_reason": "stop"
      }
    ],
    "usage": {
      "prompt_tokens": 10,
      "completion_tokens": 12,
      "total_tokens": 22
    },
    "model": "cursor-agent",
    "created": 1677858242
  }
}
```

#### Model Stream Chunk (for streaming)

```json
{
  "type": "model_stream_chunk",
  "id": "req-uuid-1234",
  "chunk_index": 0,
  "delta": {
    "role": "assistant",
    "content": "Hello"
  },
  "finish_reason": null
}
```

#### Error Response

```json
{
  "type": "model_result",
  "id": "req-uuid-1234",
  "status": "error",
  "error": {
    "code": "timeout",
    "message": "Agent response timeout after 30 seconds",
    "details": {
      "timeout_seconds": 30,
      "agent_id": "agent-cursor-1"
    }
  }
}
```

---

## 📊 Rate Limiting

### Rate Limit Headers

```http
X-RateLimit-Limit: 100           # Requests per session limit
X-RateLimit-Remaining: 85        # Remaining requests
X-RateLimit-Reset: 1677858542    # Reset timestamp
X-RateLimit-Window: 300          # Window duration (seconds)
```

### Rate Limit Response

```json
{
  "error": {
    "message": "Rate limit exceeded",
    "type": "rate_limit_exceeded",
    "code": "too_many_requests",
    "retry_after": 30,
    "limit": 100,
    "window_seconds": 300,
    "reset_at": "2024-01-01T12:05:00Z"
  }
}
```

---

## 🔒 Security Headers

### Request Headers

```http
X-MCP-Session: sess-abc123       # Optional session selection
X-Forwarded-For: 127.0.0.1       # Must be localhost
User-Agent: VoidEditor/1.0        # Client identification
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

- **Request timeout**: 30 seconds
- **Streaming timeout**: 60 seconds per chunk
- **Health check timeout**: 5 seconds
- **Session registration timeout**: 10 seconds

### Limits

- **Maximum request size**: 10MB
- **Maximum response size**: 50MB
- **Maximum concurrent requests**: 10 per session
- **Maximum message history**: 100 messages
- **Maximum token length**: 8192 tokens per message

### Response Times (SLA)

- **Health check**: < 100ms
- **Session registration**: < 500ms
- **Chat completion**: < 10s (99th percentile)
- **Streaming first chunk**: < 2s

---

**Next**: See [DEPLOYMENT.md](DEPLOYMENT.md) for security and deployment guidelines
