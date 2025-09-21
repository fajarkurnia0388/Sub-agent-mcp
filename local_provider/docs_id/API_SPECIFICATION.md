# MCP Local Provider — Spesifikasi API

> **Bahasa**: [🇺🇸 English](../API_SPECIFICATION.md) | [🇮🇩 Bahasa Indonesia](API_SPECIFICATION.md)

**Dokumentasi API lengkap untuk sistem MCP Local Provider**

## 📡 Ringkasan API

MCP Local Provider mengekspos API yang kompatibel dengan:

1. **OpenAI API** - Endpoint `/v1/chat/completions` dan `/v1/models`
2. **Ollama API** - Endpoint `/api/chat` dan `/api/tags`
3. **Management API** - Endpoint untuk monitoring dan administrasi
4. **Health API** - Endpoint untuk health check dan metrics

---

## 🌐 OpenAI Compatible API

### Base URL

```
http://127.0.0.1:8788
```

### Autentikasi

- **Required**: Header `Authorization: Bearer ${API_KEY}` untuk semua endpoint
- **Format**: API key harus berupa string minimal 32 karakter
- **Security**: API key tidak boleh di-commit ke version control

---

## 💬 Chat Completions API

### `POST /v1/chat/completions`

**Membuat completion chat dengan model agen lokal**

#### Format Request

```http
POST /v1/chat/completions HTTP/1.1
Content-Type: application/json
Authorization: Bearer your-api-key

{
  "model": "cursor-agent-local",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful coding assistant."
    },
    {
      "role": "user",
      "content": "Explain how to implement a binary search in Python"
    }
  ],
  "stream": true,
  "max_tokens": 2048,
  "temperature": 0.7,
  "top_p": 1.0,
  "frequency_penalty": 0.0,
  "presence_penalty": 0.0
}
```

#### Parameter Request

| Parameter           | Type    | Required | Default | Deskripsi                                     |
| ------------------- | ------- | -------- | ------- | --------------------------------------------- |
| `model`             | string  | Ya       | -       | Model identifier (harus "cursor-agent-local") |
| `messages`          | array   | Ya       | -       | Array pesan percakapan                        |
| `stream`            | boolean | Tidak    | false   | Apakah respons di-stream secara real-time     |
| `max_tokens`        | integer | Tidak    | 2048    | Token maksimal yang akan di-generate          |
| `temperature`       | number  | Tidak    | 0.7     | Kontrol randomness (0.0-2.0)                  |
| `top_p`             | number  | Tidak    | 1.0     | Nucleus sampling parameter                    |
| `frequency_penalty` | number  | Tidak    | 0.0     | Penalti untuk token yang sering muncul        |
| `presence_penalty`  | number  | Tidak    | 0.0     | Penalti untuk token yang sudah ada            |

#### Format Pesan

```typescript
interface ChatMessage {
  role: "system" | "user" | "assistant";
  content: string;
  name?: string; // Opsional, nama pembicara
}
```

#### Respons Non-Streaming

````json
{
  "id": "chatcmpl-123e4567-e89b-12d3-a456-426614174000",
  "object": "chat.completion",
  "created": 1704110400,
  "model": "cursor-agent-local",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Binary search adalah algoritma pencarian yang efisien untuk array yang sudah terurut. Berikut implementasinya:\n\n```python\ndef binary_search(arr, target):\n    left, right = 0, len(arr) - 1\n    \n    while left <= right:\n        mid = (left + right) // 2\n        \n        if arr[mid] == target:\n            return mid\n        elif arr[mid] < target:\n            left = mid + 1\n        else:\n            right = mid - 1\n    \n    return -1  # Tidak ditemukan\n```\n\nKompleksitas waktu: O(log n), sangat efisien untuk dataset besar."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 25,
    "completion_tokens": 150,
    "total_tokens": 175
  }
}
````

#### Respons Streaming

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1704110400,"model":"cursor-agent-local","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1704110400,"model":"cursor-agent-local","choices":[{"index":0,"delta":{"content":"Binary"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1704110400,"model":"cursor-agent-local","choices":[{"index":0,"delta":{"content":" search"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1704110400,"model":"cursor-agent-local","choices":[{"index":0,"delta":{"content":" adalah"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1704110400,"model":"cursor-agent-local","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

#### Error Responses

**400 Bad Request**

```json
{
  "error": {
    "message": "Invalid request format",
    "type": "invalid_request_error",
    "param": "messages",
    "code": "invalid_format"
  }
}
```

**401 Unauthorized**

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "authentication_error",
    "code": "invalid_api_key"
  }
}
```

**429 Rate Limit Exceeded**

```json
{
  "error": {
    "message": "Rate limit exceeded",
    "type": "rate_limit_error",
    "code": "rate_limit_exceeded",
    "param": null
  }
}
```

**503 Service Unavailable**

```json
{
  "error": {
    "message": "Agent temporarily unavailable",
    "type": "service_error",
    "code": "agent_unavailable"
  }
}
```

---

## 🧠 Models API

### `GET /v1/models`

**Mendapatkan daftar model yang tersedia**

#### Format Request

```http
GET /v1/models HTTP/1.1
Authorization: Bearer your-api-key
```

#### Respons

```json
{
  "object": "list",
  "data": [
    {
      "id": "cursor-agent-local",
      "object": "model",
      "created": 1704110400,
      "owned_by": "cursor-local-provider",
      "permission": [
        {
          "id": "modelperm-123",
          "object": "model_permission",
          "created": 1704110400,
          "allow_create_engine": false,
          "allow_sampling": true,
          "allow_logprobs": true,
          "allow_search_indices": false,
          "allow_view": true,
          "allow_fine_tuning": false,
          "organization": "*",
          "group": null,
          "is_blocking": false
        }
      ],
      "root": "cursor-agent-local",
      "parent": null
    }
  ]
}
```

---

## 🦙 Ollama Compatible API

### `POST /api/chat`

**Chat endpoint kompatibel Ollama**

#### Format Request

```json
{
  "model": "cursor-agent-local",
  "messages": [
    {
      "role": "user",
      "content": "Explain Python decorators"
    }
  ],
  "stream": true
}
```

#### Respons Streaming

```json
{"model":"cursor-agent-local","created_at":"2024-01-01T12:00:00.000Z","message":{"role":"assistant","content":"Python"},"done":false}
{"model":"cursor-agent-local","created_at":"2024-01-01T12:00:00.100Z","message":{"role":"assistant","content":" decorators"},"done":false}
{"model":"cursor-agent-local","created_at":"2024-01-01T12:00:00.200Z","message":{"role":"assistant","content":" adalah"},"done":false}
{"model":"cursor-agent-local","created_at":"2024-01-01T12:00:00.300Z","message":{"role":"assistant","content":""},"done":true}
```

### `GET /api/tags`

**Mendapatkan daftar model (format Ollama)**

#### Respons

```json
{
  "models": [
    {
      "name": "cursor-agent-local",
      "modified_at": "2024-01-01T12:00:00.000Z",
      "size": 0,
      "digest": "sha256:1234567890abcdef",
      "details": {
        "format": "cursor-agent",
        "family": "local-provider",
        "families": ["local-provider"],
        "parameter_size": "7B",
        "quantization_level": "Q4_0"
      }
    }
  ]
}
```

---

## 🔧 Management API

### `GET /health`

**Health check endpoint**

#### Respons

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T12:00:00.000Z",
  "version": "1.0.0",
  "uptime_seconds": 3600,
  "active_sessions": 3,
  "mcp_connection": {
    "status": "connected",
    "last_ping": "2024-01-01T11:59:55.000Z"
  }
}
```

### `GET /metrics`

**Performance metrics**

#### Respons

```json
{
  "sessions": {
    "active": 3,
    "total_created": 150,
    "total_expired": 147
  },
  "requests": {
    "total": 1250,
    "successful": 1200,
    "failed": 50,
    "rate_limited": 25
  },
  "performance": {
    "avg_response_time_ms": 250,
    "p95_response_time_ms": 500,
    "p99_response_time_ms": 1000,
    "memory_usage_mb": 128,
    "cpu_usage_percent": 15.5
  },
  "tokens": {
    "total_processed": 500000,
    "avg_tokens_per_request": 400,
    "tokens_per_second": 150
  }
}
```

### `GET /config`

**Konfigurasi saat ini (authenticated)**

#### Headers

```http
Authorization: Bearer your-api-key
```

#### Respons

```json
{
  "provider": {
    "model_name": "cursor-agent-local",
    "max_tokens": 2048,
    "temperature": 0.7,
    "stream_enabled": true
  },
  "security": {
    "rate_limit_rpm": 60,
    "rate_limit_tpm": 100000,
    "session_ttl_seconds": 300,
    "max_sessions": 10
  },
  "mcp": {
    "connection_status": "connected",
    "agent_id": "local-provider-1",
    "heartbeat_interval": 30
  }
}
```

---

## 📊 Rate Limiting

### Rate Limit Headers

Semua respons menyertakan header rate limiting:

```http
X-RateLimit-Limit-Requests: 60         # Requests per minute
X-RateLimit-Remaining-Requests: 45     # Remaining requests
X-RateLimit-Reset-Requests: 1704110460 # Reset timestamp

X-RateLimit-Limit-Tokens: 100000       # Tokens per minute
X-RateLimit-Remaining-Tokens: 85000    # Remaining tokens
X-RateLimit-Reset-Tokens: 1704110460   # Reset timestamp
```

### Rate Limit Tiers

| Tier            | Requests/Min | Tokens/Min | Concurrent Sessions |
| --------------- | ------------ | ---------- | ------------------- |
| **Default**     | 60           | 100,000    | 5                   |
| **Development** | 120          | 200,000    | 10                  |
| **Production**  | 300          | 500,000    | 25                  |

---

## 🔒 Security Headers

### Request Headers

```http
Authorization: Bearer your-api-key      # Required for all endpoints
Content-Type: application/json         # For POST requests
User-Agent: VoidEditor/1.0             # Client identification
X-Forwarded-For: 127.0.0.1            # Must be localhost
```

### Response Headers

```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Access-Control-Allow-Origin: http://127.0.0.1:*
Content-Security-Policy: default-src 'self'
X-RateLimit-*: (rate limiting headers)
```

---

## 🔄 WebSocket API (Internal MCP)

### Connection

```
ws://127.0.0.1:8788/mcp/agent
```

### Message Types

#### Model Invoke Message

```json
{
  "type": "model_invoke",
  "id": "req-123e4567-e89b-12d3-a456-426614174000",
  "model": "cursor-agent-local",
  "messages": [{ "role": "user", "content": "Hello" }],
  "stream": true,
  "max_tokens": 2048,
  "temperature": 0.7
}
```

#### Model Result Message

```json
{
  "type": "model_result",
  "id": "req-123e4567-e89b-12d3-a456-426614174000",
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
  }
}
```

#### Model Stream Chunk

```json
{
  "type": "model_stream_chunk",
  "id": "req-123e4567-e89b-12d3-a456-426614174000",
  "chunk": {
    "id": "chatcmpl-123",
    "object": "chat.completion.chunk",
    "created": 1704110400,
    "model": "cursor-agent-local",
    "choices": [
      {
        "index": 0,
        "delta": { "content": "Hello" },
        "finish_reason": null
      }
    ]
  },
  "finished": false
}
```

---

## ⚡ Performance Specifications

### Timeouts

- **HTTP Request timeout**: 30 detik
- **WebSocket connection timeout**: 10 detik
- **MCP message timeout**: 5 detik
- **Session cleanup interval**: 60 detik

### Response Time SLA

- **Health check**: < 100ms
- **Model list**: < 200ms
- **Chat completion (first token)**: < 1s
- **Streaming response**: < 50ms per chunk

### Limits

- **Maximum request size**: 1MB
- **Maximum response size**: 10MB
- **Maximum concurrent connections**: 100
- **Maximum session duration**: 1 jam
- **Maximum tokens per request**: 8192

---

## 🐛 Error Codes

### HTTP Status Codes

| Code | Deskripsi                                  |
| ---- | ------------------------------------------ |
| 200  | OK - Request berhasil                      |
| 400  | Bad Request - Format request tidak valid   |
| 401  | Unauthorized - API key tidak valid         |
| 403  | Forbidden - Akses ditolak                  |
| 429  | Too Many Requests - Rate limit terlampaui  |
| 500  | Internal Server Error - Error server       |
| 502  | Bad Gateway - MCP connection error         |
| 503  | Service Unavailable - Agent tidak tersedia |

### Error Types

| Type                    | Deskripsi                  |
| ----------------------- | -------------------------- |
| `invalid_request_error` | Format request tidak valid |
| `authentication_error`  | Error autentikasi          |
| `rate_limit_error`      | Rate limit terlampaui      |
| `service_error`         | Error layanan internal     |
| `mcp_error`             | Error komunikasi MCP       |

---

## 📝 Usage Examples

### Contoh cURL

```bash
# Chat completion request
curl -X POST http://127.0.0.1:8788/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{
    "model": "cursor-agent-local",
    "messages": [
      {"role": "user", "content": "Explain Python list comprehensions"}
    ],
    "stream": false,
    "max_tokens": 500
  }'

# Streaming completion
curl -X POST http://127.0.0.1:8788/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{
    "model": "cursor-agent-local",
    "messages": [
      {"role": "user", "content": "Write a Python function"}
    ],
    "stream": true
  }' \
  --no-buffer

# Get models
curl -X GET http://127.0.0.1:8788/v1/models \
  -H "Authorization: Bearer your-api-key"

# Health check
curl -X GET http://127.0.0.1:8788/health
```

### Contoh JavaScript

```javascript
// Using fetch API
async function chatCompletion(message) {
  const response = await fetch("http://127.0.0.1:8788/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: "Bearer your-api-key",
    },
    body: JSON.stringify({
      model: "cursor-agent-local",
      messages: [{ role: "user", content: message }],
      stream: false,
    }),
  });

  const data = await response.json();
  return data.choices[0].message.content;
}

// Streaming example
async function streamingChat(message) {
  const response = await fetch("http://127.0.0.1:8788/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: "Bearer your-api-key",
    },
    body: JSON.stringify({
      model: "cursor-agent-local",
      messages: [{ role: "user", content: message }],
      stream: true,
    }),
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const chunk = decoder.decode(value);
    const lines = chunk.split("\n");

    for (const line of lines) {
      if (line.startsWith("data: ")) {
        const data = line.slice(6);
        if (data === "[DONE]") return;

        try {
          const parsed = JSON.parse(data);
          const content = parsed.choices[0]?.delta?.content;
          if (content) {
            console.log(content); // Process streaming content
          }
        } catch (e) {
          // Skip invalid JSON
        }
      }
    }
  }
}
```

### Contoh Python

```python
import requests
import json

class CursorAgentClient:
    def __init__(self, api_key: str, base_url: str = "http://127.0.0.1:8788"):
        self.api_key = api_key
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

    def chat_completion(self, messages: list, stream: bool = False) -> dict:
        """Send chat completion request"""
        payload = {
            "model": "cursor-agent-local",
            "messages": messages,
            "stream": stream
        }

        response = requests.post(
            f"{self.base_url}/v1/chat/completions",
            headers=self.headers,
            json=payload,
            stream=stream
        )

        if stream:
            return self._handle_stream(response)
        else:
            return response.json()

    def _handle_stream(self, response):
        """Handle streaming response"""
        for line in response.iter_lines():
            if line:
                line = line.decode('utf-8')
                if line.startswith('data: '):
                    data = line[6:]
                    if data == '[DONE]':
                        break
                    try:
                        yield json.loads(data)
                    except json.JSONDecodeError:
                        continue

# Usage
client = CursorAgentClient("your-api-key")

# Non-streaming
result = client.chat_completion([
    {"role": "user", "content": "Explain async/await in Python"}
])
print(result['choices'][0]['message']['content'])

# Streaming
for chunk in client.chat_completion([
    {"role": "user", "content": "Write a web scraper"}
], stream=True):
    content = chunk['choices'][0]['delta'].get('content', '')
    print(content, end='', flush=True)
```

---

**Next**: Lihat [DEPLOYMENT.md](DEPLOYMENT.md) untuk panduan deployment dan konfigurasi keamanan
