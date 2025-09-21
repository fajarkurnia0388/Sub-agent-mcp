# MCP Local Provider — Panduan Testing

> **Bahasa**: [🇺🇸 English](../TESTING.md) | [🇮🇩 Bahasa Indonesia](TESTING.md)

**Strategi testing komprehensif untuk sistem MCP Local Provider**

## 🧪 Gambaran Testing

Sistem MCP Local Provider memerlukan testing komprehensif di berbagai layer: unit tests, integration tests, security tests, performance tests, dan end-to-end testing scenarios.

### Filosofi Testing

1. **Security First** - Semua fitur keamanan harus di-test secara menyeluruh
2. **API Compatibility** - Kompatibilitas OpenAI dan Ollama API
3. **MCP Communication** - Komunikasi dengan agen Cursor
4. **Performance** - Rate limiting dan scalability testing
5. **Real-world Scenarios** - Test dengan use cases nyata

---

## 🔧 Setup Testing

### Environment Testing

```bash
# Buat test environment
python -m venv test_env
source test_env/bin/activate  # Windows: test_env\Scripts\activate

# Install test dependencies
pip install pytest pytest-asyncio pytest-httpx httpx faker freezegun

# Install provider package dalam development mode
pip install -e .
```

### Konfigurasi Testing (`tests/config/test_config.yaml`)

```yaml
provider:
  host: "127.0.0.1"
  port: 8789 # Port berbeda untuk testing
  model_name: "cursor-agent-local"
  debug: true

security:
  api_key: "test-api-key-for-testing-only"
  session_ttl: 60 # Lebih pendek untuk testing
  max_sessions: 5

rate_limiting:
  enabled: true
  requests_per_minute: 120 # Limit lebih tinggi untuk testing
  tokens_per_minute: 200000

logging:
  level: "DEBUG"
  handlers: ["console"]

sessions:
  cleanup_interval: 10 # Cleanup lebih cepat
  max_idle_time: 60

mcp:
  bridge_url: "ws://127.0.0.1:8787"
  agent_id: "test-provider-1"
  connection_timeout: 10
```

### Test Fixtures (`tests/conftest.py`)

```python
import pytest
import asyncio
from fastapi.testclient import TestClient
from httpx import AsyncClient
import tempfile
import os
from datetime import datetime, timedelta
from unittest.mock import AsyncMock, MagicMock

from local_provider.adapter import app
from local_provider.session_manager import SessionManager
from local_provider.mcp_client import MCPClient
from local_provider.security import SecurityManager

@pytest.fixture(scope="session")
def event_loop():
    """Buat event loop untuk async tests"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def test_client():
    """FastAPI test client"""
    return TestClient(app)

@pytest.fixture
async def async_client():
    """Async HTTP client untuk testing"""
    async with AsyncClient(app=app, base_url="http://testserver") as client:
        yield client

@pytest.fixture
def test_config():
    """Test configuration"""
    return {
        "provider": {"host": "127.0.0.1", "port": 8789, "debug": True},
        "security": {"api_key": "test-api-key", "session_ttl": 60},
        "rate_limiting": {"requests_per_minute": 120, "tokens_per_minute": 200000},
        "sessions": {"cleanup_interval": 10, "max_idle_time": 60},
        "mcp": {"bridge_url": "ws://127.0.0.1:8787", "agent_id": "test-provider-1"}
    }

@pytest.fixture
def session_manager(test_config):
    """Session manager instance"""
    return SessionManager(test_config["sessions"])

@pytest.fixture
def mock_mcp_client():
    """Mock MCP client"""
    client = AsyncMock(spec=MCPClient)
    client.is_connected.return_value = True
    client.get_connection_status.return_value = {
        "status": "connected",
        "last_ping": datetime.utcnow().isoformat(),
        "messages_sent": 0,
        "messages_received": 0
    }
    return client

@pytest.fixture
def security_manager(test_config):
    """Security manager instance"""
    return SecurityManager(test_config["security"])

@pytest.fixture
def valid_chat_request():
    """Valid chat completion request"""
    return {
        "model": "cursor-agent-local",
        "messages": [
            {"role": "user", "content": "Hello, test message"}
        ],
        "stream": False,
        "max_tokens": 100,
        "temperature": 0.7
    }

@pytest.fixture
def auth_headers():
    """Authentication headers untuk API requests"""
    return {"Authorization": "Bearer test-api-key"}

@pytest.fixture
def mock_agent_response():
    """Mock response dari agent"""
    return {
        "type": "model_result",
        "id": "test-request-id",
        "choices": [
            {
                "index": 0,
                "message": {
                    "role": "assistant",
                    "content": "Hello! This is a test response from the agent."
                },
                "finish_reason": "stop"
            }
        ],
        "usage": {
            "prompt_tokens": 10,
            "completion_tokens": 15,
            "total_tokens": 25
        }
    }
```

---

## 🧪 Unit Tests

### Security Tests (`tests/unit/test_security.py`)

```python
import pytest
from local_provider.security import SecurityManager

class TestSecurityManager:

    def test_verify_api_key_valid(self, security_manager):
        """Test API key verification dengan valid key"""
        assert security_manager.verify_api_key("test-api-key") == True

    def test_verify_api_key_invalid(self, security_manager):
        """Test API key verification dengan invalid key"""
        assert security_manager.verify_api_key("wrong-api-key") == False

    def test_generate_session_id(self, security_manager):
        """Test session ID generation"""
        session_id = security_manager.generate_session_id()
        assert session_id.startswith("sess-")
        assert len(session_id) > 10

        # Test uniqueness
        session_id2 = security_manager.generate_session_id()
        assert session_id != session_id2

    def test_validate_request_data_valid(self, security_manager):
        """Test request data validation dengan data valid"""
        valid_data = {
            "model": "cursor-agent-local",
            "messages": [
                {"role": "user", "content": "Test message"}
            ]
        }
        assert security_manager.validate_request_data(valid_data) == True

    def test_validate_request_data_invalid_model(self, security_manager):
        """Test request data validation dengan model invalid"""
        invalid_data = {
            "model": "invalid-model",
            "messages": [
                {"role": "user", "content": "Test message"}
            ]
        }
        assert security_manager.validate_request_data(invalid_data) == False

    def test_validate_request_data_invalid_messages(self, security_manager):
        """Test request data validation dengan messages invalid"""
        invalid_data = {
            "model": "cursor-agent-local",
            "messages": [
                {"invalid": "format"}
            ]
        }
        assert security_manager.validate_request_data(invalid_data) == False

    def test_sanitize_content(self, security_manager):
        """Test content sanitization"""
        # Test script removal
        malicious_content = "Hello <script>alert('xss')</script> world"
        sanitized = security_manager.sanitize_content(malicious_content)
        assert "<script>" not in sanitized
        assert "Hello  world" in sanitized

        # Test length limiting
        long_content = "x" * 200000
        sanitized = security_manager.sanitize_content(long_content)
        assert len(sanitized) <= 100000
```

### Session Management Tests (`tests/unit/test_session_manager.py`)

```python
import pytest
import asyncio
from datetime import datetime, timedelta
from freezegun import freeze_time

from local_provider.session_manager import SessionManager

class TestSessionManager:

    @pytest.mark.asyncio
    async def test_create_session(self, session_manager):
        """Test session creation"""
        session = await session_manager.create_session(
            client_ip="127.0.0.1",
            model="cursor-agent-local",
            max_tokens=2048
        )

        assert session.session_id.startswith("sess-")
        assert session.client_ip == "127.0.0.1"
        assert session.model == "cursor-agent-local"
        assert session.max_tokens == 2048
        assert session.request_count == 0

    @pytest.mark.asyncio
    async def test_get_session_valid(self, session_manager):
        """Test getting valid session"""
        # Create session
        session = await session_manager.create_session(
            "127.0.0.1", "cursor-agent-local"
        )

        # Get session
        retrieved = await session_manager.get_session(session.session_id)
        assert retrieved is not None
        assert retrieved.session_id == session.session_id

    @pytest.mark.asyncio
    async def test_get_session_invalid(self, session_manager):
        """Test getting non-existent session"""
        result = await session_manager.get_session("sess-invalid")
        assert result is None

    @pytest.mark.asyncio
    async def test_session_expiration(self, session_manager):
        """Test session expiration"""
        # Create session dengan TTL pendek
        session_manager.session_ttl = 1
        session = await session_manager.create_session(
            "127.0.0.1", "cursor-agent-local"
        )

        # Session should be valid initially
        retrieved = await session_manager.get_session(session.session_id)
        assert retrieved is not None

        # Wait untuk expiration
        await asyncio.sleep(2)

        # Session should be expired
        expired = await session_manager.get_session(session.session_id)
        assert expired is None

    @pytest.mark.asyncio
    async def test_max_sessions_limit(self, session_manager):
        """Test maximum sessions limit"""
        # Set limit rendah untuk testing
        session_manager.max_sessions = 2

        # Create dua sessions
        session1 = await session_manager.create_session("127.0.0.1", "cursor-agent-local")
        session2 = await session_manager.create_session("127.0.0.2", "cursor-agent-local")

        # Third session should fail
        with pytest.raises(ValueError, match="Maximum sessions reached"):
            await session_manager.create_session("127.0.0.3", "cursor-agent-local")

    @pytest.mark.asyncio
    async def test_cleanup_expired_sessions(self, session_manager):
        """Test cleanup dari expired sessions"""
        # Create sessions dengan TTL berbeda
        session_manager.session_ttl = 60

        session1 = await session_manager.create_session("127.0.0.1", "cursor-agent-local")

        # Manually expire session
        session1.expires_at = datetime.utcnow() - timedelta(seconds=10)

        # Run cleanup
        await session_manager._cleanup_expired_sessions()

        # Session should be removed
        assert session1.session_id not in session_manager.sessions
```

### Rate Limiter Tests (`tests/unit/test_rate_limiter.py`)

```python
import pytest
import asyncio
import time
from local_provider.rate_limiter import RateLimiter

class TestRateLimiter:

    @pytest.mark.asyncio
    async def test_request_within_limit(self):
        """Test request dalam batas limit"""
        limiter = RateLimiter(requests_per_minute=10, tokens_per_minute=1000)

        allowed, info = await limiter.check_rate_limit("client1", 50)
        assert allowed == True
        assert info["requests_remaining"] == 9
        assert info["tokens_remaining"] == 950

    @pytest.mark.asyncio
    async def test_request_limit_exceeded(self):
        """Test request melebihi limit"""
        limiter = RateLimiter(requests_per_minute=2, tokens_per_minute=1000)

        # First two requests should pass
        await limiter.check_rate_limit("client1", 10)
        await limiter.check_rate_limit("client1", 10)

        # Third request should fail
        allowed, info = await limiter.check_rate_limit("client1", 10)
        assert allowed == False
        assert info["error"] == "Request rate limit exceeded"

    @pytest.mark.asyncio
    async def test_token_limit_exceeded(self):
        """Test token melebihi limit"""
        limiter = RateLimiter(requests_per_minute=10, tokens_per_minute=100)

        # Request dengan banyak tokens
        allowed, info = await limiter.check_rate_limit("client1", 150)
        assert allowed == False
        assert info["error"] == "Token rate limit exceeded"

    @pytest.mark.asyncio
    async def test_different_clients_separate_limits(self):
        """Test bahwa client berbeda punya limit terpisah"""
        limiter = RateLimiter(requests_per_minute=2, tokens_per_minute=1000)

        # Client1 exhausts limit
        await limiter.check_rate_limit("client1", 10)
        await limiter.check_rate_limit("client1", 10)

        # Client2 should still be allowed
        allowed, info = await limiter.check_rate_limit("client2", 10)
        assert allowed == True

    @pytest.mark.asyncio
    async def test_rate_limit_window_reset(self):
        """Test reset rate limit window"""
        limiter = RateLimiter(requests_per_minute=2, tokens_per_minute=1000)

        # Exhaust limit
        await limiter.check_rate_limit("client1", 10)
        await limiter.check_rate_limit("client1", 10)

        # Should be blocked
        allowed, _ = await limiter.check_rate_limit("client1", 10)
        assert allowed == False

        # Simulate time passing (wait for window reset)
        # Dalam real test, bisa menggunakan mock time
        await asyncio.sleep(61)

        # Should be allowed again
        allowed, _ = await limiter.check_rate_limit("client1", 10)
        assert allowed == True
```

---

## 🔗 Integration Tests

### API Integration Tests (`tests/integration/test_api.py`)

```python
import pytest
import json
from httpx import AsyncClient

class TestProviderAPI:

    @pytest.mark.asyncio
    async def test_health_check(self, async_client):
        """Test health check endpoint"""
        response = await async_client.get("/health")
        assert response.status_code == 200

        data = response.json()
        assert data["status"] == "healthy"
        assert "timestamp" in data
        assert "version" in data
        assert "active_sessions" in data

    @pytest.mark.asyncio
    async def test_list_models(self, async_client, auth_headers):
        """Test models list endpoint"""
        response = await async_client.get("/v1/models", headers=auth_headers)
        assert response.status_code == 200

        data = response.json()
        assert data["object"] == "list"
        assert len(data["data"]) == 1
        assert data["data"][0]["id"] == "cursor-agent-local"

    @pytest.mark.asyncio
    async def test_chat_completion_non_streaming(
        self, async_client, valid_chat_request, auth_headers, mock_mcp_client, mock_agent_response
    ):
        """Test non-streaming chat completion"""
        # Mock MCP client response
        mock_mcp_client.send_message.return_value = mock_agent_response

        response = await async_client.post(
            "/v1/chat/completions",
            json=valid_chat_request,
            headers=auth_headers
        )

        assert response.status_code == 200
        data = response.json()

        assert data["object"] == "chat.completion"
        assert data["model"] == "cursor-agent-local"
        assert len(data["choices"]) == 1
        assert data["choices"][0]["message"]["role"] == "assistant"
        assert "usage" in data

    @pytest.mark.asyncio
    async def test_chat_completion_streaming(
        self, async_client, auth_headers, mock_mcp_client
    ):
        """Test streaming chat completion"""
        streaming_request = {
            "model": "cursor-agent-local",
            "messages": [{"role": "user", "content": "Hello"}],
            "stream": True
        }

        # Mock streaming responses
        async def mock_stream_responses(request_id):
            chunks = [
                {
                    "type": "model_stream_chunk",
                    "id": request_id,
                    "chunk": {"delta": {"content": "Hello"}},
                    "finished": False
                },
                {
                    "type": "model_stream_chunk",
                    "id": request_id,
                    "chunk": {"delta": {"content": " there!"}},
                    "finished": True
                }
            ]
            for chunk in chunks:
                yield chunk

        mock_mcp_client.stream_responses.return_value = mock_stream_responses("test-id")

        response = await async_client.post(
            "/v1/chat/completions",
            json=streaming_request,
            headers=auth_headers
        )

        assert response.status_code == 200
        assert response.headers["content-type"] == "text/event-stream; charset=utf-8"

    @pytest.mark.asyncio
    async def test_invalid_api_key(self, async_client, valid_chat_request):
        """Test dengan invalid API key"""
        invalid_headers = {"Authorization": "Bearer invalid-key"}

        response = await async_client.post(
            "/v1/chat/completions",
            json=valid_chat_request,
            headers=invalid_headers
        )

        assert response.status_code == 401
        data = response.json()
        assert "Invalid API key" in data["detail"]

    @pytest.mark.asyncio
    async def test_invalid_model(self, async_client, auth_headers):
        """Test dengan model invalid"""
        invalid_request = {
            "model": "invalid-model",
            "messages": [{"role": "user", "content": "Test"}]
        }

        response = await async_client.post(
            "/v1/chat/completions",
            json=invalid_request,
            headers=auth_headers
        )

        assert response.status_code == 422  # Validation error

    @pytest.mark.asyncio
    async def test_rate_limiting(self, async_client, auth_headers):
        """Test rate limiting behavior"""
        # Buat banyak requests cepat-cepat
        requests = []
        for i in range(25):  # Melebihi limit 20 requests
            request = {
                "model": "cursor-agent-local",
                "messages": [{"role": "user", "content": f"Message {i}"}],
                "max_tokens": 10
            }
            requests.append(
                async_client.post("/v1/chat/completions", json=request, headers=auth_headers)
            )

        # Execute semua requests secara parallel
        responses = await asyncio.gather(*requests, return_exceptions=True)

        # Hitung status codes
        success_count = sum(1 for r in responses if hasattr(r, 'status_code') and r.status_code == 200)
        rate_limited_count = sum(1 for r in responses if hasattr(r, 'status_code') and r.status_code == 429)

        # Should have some rate limited responses
        assert rate_limited_count > 0
        assert success_count < len(requests)
```

### Ollama API Tests (`tests/integration/test_ollama_api.py`)

```python
import pytest
import json
from httpx import AsyncClient

class TestOllamaAPI:

    @pytest.mark.asyncio
    async def test_ollama_chat_non_streaming(self, async_client, auth_headers, mock_agent_response):
        """Test Ollama chat endpoint non-streaming"""
        ollama_request = {
            "model": "cursor-agent-local",
            "messages": [
                {"role": "user", "content": "Hello from Ollama"}
            ],
            "stream": False
        }

        response = await async_client.post(
            "/api/chat",
            json=ollama_request,
            headers=auth_headers
        )

        assert response.status_code == 200
        data = response.json()

        assert data["model"] == "cursor-agent-local"
        assert "created_at" in data
        assert data["message"]["role"] == "assistant"
        assert data["done"] == True

    @pytest.mark.asyncio
    async def test_ollama_tags(self, async_client, auth_headers):
        """Test Ollama tags endpoint"""
        response = await async_client.get("/api/tags", headers=auth_headers)
        assert response.status_code == 200

        data = response.json()
        assert "models" in data
        assert len(data["models"]) == 1
        assert data["models"][0]["name"] == "cursor-agent-local"

    @pytest.mark.asyncio
    async def test_ollama_chat_streaming(self, async_client, auth_headers, mock_mcp_client):
        """Test Ollama streaming chat"""
        ollama_request = {
            "model": "cursor-agent-local",
            "messages": [{"role": "user", "content": "Stream test"}],
            "stream": True
        }

        # Mock streaming response
        async def mock_stream():
            yield {
                "type": "model_stream_chunk",
                "chunk": {"delta": {"content": "Hello"}},
                "finished": False
            }
            yield {
                "type": "model_stream_chunk",
                "chunk": {"delta": {"content": " World!"}},
                "finished": True
            }

        mock_mcp_client.stream_responses.return_value = mock_stream()

        response = await async_client.post(
            "/api/chat",
            json=ollama_request,
            headers=auth_headers
        )

        assert response.status_code == 200
        assert response.headers["content-type"] == "application/json; charset=utf-8"
```

---

## 🔒 Security Tests

### Authentication Tests (`tests/security/test_authentication.py`)

```python
import pytest
import time
from local_provider.security import SecurityManager

class TestSecurityScenarios:

    def test_api_key_timing_attack_resistance(self, security_manager):
        """Test resistance terhadap timing attacks pada API key verification"""
        valid_key = "test-api-key"
        invalid_key = "wrong-api-key"

        # Measure timing untuk valid key
        start = time.perf_counter()
        security_manager.verify_api_key(valid_key)
        valid_time = time.perf_counter() - start

        # Measure timing untuk invalid key
        start = time.perf_counter()
        security_manager.verify_api_key(invalid_key)
        invalid_time = time.perf_counter() - start

        # Times should be similar (dalam reasonable variance)
        time_diff = abs(valid_time - invalid_time)
        assert time_diff < 0.001  # Kurang dari 1ms difference

    def test_session_id_uniqueness(self, security_manager):
        """Test uniqueness dari generated session IDs"""
        session_ids = set()

        # Generate banyak session IDs
        for _ in range(1000):
            session_id = security_manager.generate_session_id()
            assert session_id not in session_ids
            session_ids.add(session_id)

        # All should be unique
        assert len(session_ids) == 1000

    def test_content_sanitization_xss(self, security_manager):
        """Test XSS prevention dalam content sanitization"""
        malicious_inputs = [
            "<script>alert('xss')</script>",
            "javascript:alert('xss')",
            "<img src=x onerror=alert('xss')>",
            "<svg onload=alert('xss')>",
            "<iframe src=javascript:alert('xss')></iframe>"
        ]

        for malicious_input in malicious_inputs:
            sanitized = security_manager.sanitize_content(malicious_input)
            assert "script" not in sanitized.lower()
            assert "javascript:" not in sanitized.lower()
            assert "onerror" not in sanitized.lower()
            assert "onload" not in sanitized.lower()

    def test_content_length_limiting(self, security_manager):
        """Test content length limiting"""
        # Test dengan content yang sangat panjang
        long_content = "x" * 200000  # 200KB
        sanitized = security_manager.sanitize_content(long_content)

        # Should be limited ke 100KB
        assert len(sanitized) <= 100000
```

### Input Validation Tests (`tests/security/test_validation.py`)

```python
import pytest

class TestInputValidation:

    @pytest.mark.parametrize("invalid_model", [
        "",  # Empty
        "gpt-4",  # Wrong model
        "claude-3",  # Wrong model
        "../../../etc/passwd",  # Path traversal
        "model with spaces",  # Spaces
        "model@invalid",  # Special characters
    ])
    @pytest.mark.asyncio
    async def test_invalid_model_names(self, async_client, auth_headers, invalid_model):
        """Test rejection dari invalid model names"""
        request_data = {
            "model": invalid_model,
            "messages": [{"role": "user", "content": "Test"}]
        }

        response = await async_client.post(
            "/v1/chat/completions",
            json=request_data,
            headers=auth_headers
        )
        assert response.status_code == 422  # Validation error

    @pytest.mark.parametrize("invalid_message", [
        [],  # Empty messages
        [{"content": "missing role"}],  # Missing role
        [{"role": "missing content"}],  # Missing content
        [{"role": "invalid_role", "content": "test"}],  # Invalid role
        [{"role": "user", "content": "x" * 200000}],  # Too long content
    ])
    @pytest.mark.asyncio
    async def test_invalid_messages(self, async_client, auth_headers, invalid_message):
        """Test rejection dari invalid message formats"""
        request_data = {
            "model": "cursor-agent-local",
            "messages": invalid_message
        }

        response = await async_client.post(
            "/v1/chat/completions",
            json=request_data,
            headers=auth_headers
        )
        assert response.status_code in [400, 422]  # Bad request atau validation error

    @pytest.mark.parametrize("invalid_param", [
        {"max_tokens": -1},  # Negative
        {"max_tokens": 10000},  # Too large
        {"temperature": -0.5},  # Too low
        {"temperature": 3.0},  # Too high
        {"top_p": -0.1},  # Too low
        {"top_p": 1.5},  # Too high
    ])
    @pytest.mark.asyncio
    async def test_invalid_parameters(self, async_client, auth_headers, invalid_param):
        """Test rejection dari invalid parameters"""
        request_data = {
            "model": "cursor-agent-local",
            "messages": [{"role": "user", "content": "Test"}],
            **invalid_param
        }

        response = await async_client.post(
            "/v1/chat/completions",
            json=request_data,
            headers=auth_headers
        )
        assert response.status_code == 422  # Validation error
```

---

## ⚡ Performance Tests

### Load Testing (`tests/performance/test_load.py`)

```python
import pytest
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor

class TestPerformance:

    @pytest.mark.asyncio
    async def test_concurrent_requests(self, async_client, auth_headers):
        """Test handling multiple concurrent requests"""
        async def make_request(request_id):
            request_data = {
                "model": "cursor-agent-local",
                "messages": [
                    {"role": "user", "content": f"Concurrent request {request_id}"}
                ],
                "max_tokens": 50
            }

            start_time = time.time()
            response = await async_client.post(
                "/v1/chat/completions",
                json=request_data,
                headers=auth_headers
            )
            end_time = time.time()

            return {
                "status_code": response.status_code,
                "response_time": end_time - start_time,
                "request_id": request_id
            }

        # Create 20 concurrent requests
        tasks = [make_request(i) for i in range(20)]
        results = await asyncio.gather(*tasks)

        # Analyze results
        success_count = sum(1 for r in results if r["status_code"] == 200)
        avg_response_time = sum(r["response_time"] for r in results) / len(results)

        # Most requests should succeed
        assert success_count >= 15  # Allow some rate limiting

        # Average response time should be reasonable
        assert avg_response_time < 2.0  # Less than 2 seconds

    @pytest.mark.asyncio
    async def test_streaming_performance(self, async_client, auth_headers, mock_mcp_client):
        """Test streaming response performance"""

        # Mock streaming dengan many chunks
        async def mock_many_chunks(request_id):
            for i in range(100):  # 100 chunks
                yield {
                    "type": "model_stream_chunk",
                    "id": request_id,
                    "chunk": {"delta": {"content": f"chunk{i} "}},
                    "finished": i == 99
                }

        mock_mcp_client.stream_responses.return_value = mock_many_chunks("test-id")

        streaming_request = {
            "model": "cursor-agent-local",
            "messages": [{"role": "user", "content": "Stream many chunks"}],
            "stream": True
        }

        start_time = time.time()

        response = await async_client.post(
            "/v1/chat/completions",
            json=streaming_request,
            headers=auth_headers
        )

        # Count chunks
        chunk_count = 0
        async for line in response.aiter_lines():
            if line.startswith("data: ") and not line.endswith("[DONE]"):
                chunk_count += 1

        end_time = time.time()
        total_time = end_time - start_time

        assert response.status_code == 200
        assert chunk_count == 100
        assert total_time < 5.0  # Should complete within 5 seconds

    @pytest.mark.asyncio
    async def test_memory_usage_under_load(self, session_manager):
        """Test memory usage tidak tumbuh berlebihan"""
        import psutil
        import os

        process = psutil.Process(os.getpid())
        initial_memory = process.memory_info().rss

        # Create banyak sessions
        sessions = []
        for i in range(100):
            session = await session_manager.create_session(
                f"127.0.0.{i}", "cursor-agent-local"
            )
            sessions.append(session)

        peak_memory = process.memory_info().rss

        # Clean up sessions
        for session in sessions:
            await session_manager.expire_session(session.session_id)

        final_memory = process.memory_info().rss

        # Memory tidak boleh tumbuh berlebihan
        memory_growth = peak_memory - initial_memory
        memory_ratio = memory_growth / initial_memory

        assert memory_ratio < 3.0  # Kurang dari 200% growth

        # Memory should be released setelah cleanup
        cleanup_ratio = (peak_memory - final_memory) / memory_growth
        assert cleanup_ratio > 0.3  # At least 30% cleanup
```

---

## 🎯 End-to-End Tests

### Complete Flow Tests (`tests/e2e/test_complete_flow.py`)

```python
import pytest
import asyncio
import json
from unittest.mock import AsyncMock

class TestEndToEndFlows:

    @pytest.mark.asyncio
    async def test_complete_development_workflow(self, async_client, auth_headers, mock_mcp_client):
        """Test complete development workflow dengan local provider"""

        # Mock agent responses untuk berbagai jenis requests
        def create_agent_response(content, request_id="test-id"):
            return {
                "type": "model_result",
                "id": request_id,
                "choices": [
                    {
                        "index": 0,
                        "message": {"role": "assistant", "content": content},
                        "finish_reason": "stop"
                    }
                ],
                "usage": {"prompt_tokens": 20, "completion_tokens": 50, "total_tokens": 70}
            }

        # Test sequence: explain code → write code → debug → optimize
        test_scenarios = [
            {
                "name": "explain_code",
                "request": {
                    "model": "cursor-agent-local",
                    "messages": [
                        {"role": "user", "content": "Explain this Python function: def factorial(n): return 1 if n <= 1 else n * factorial(n-1)"}
                    ]
                },
                "expected_response": "This is a recursive factorial function..."
            },
            {
                "name": "write_code",
                "request": {
                    "model": "cursor-agent-local",
                    "messages": [
                        {"role": "user", "content": "Write a binary search function in Python"}
                    ]
                },
                "expected_response": "def binary_search(arr, target):"
            },
            {
                "name": "debug_code",
                "request": {
                    "model": "cursor-agent-local",
                    "messages": [
                        {"role": "user", "content": "Why is this code throwing IndexError?"}
                    ]
                },
                "expected_response": "The IndexError occurs because..."
            },
            {
                "name": "optimize_code",
                "request": {
                    "model": "cursor-agent-local",
                    "messages": [
                        {"role": "user", "content": "How can I optimize this loop for better performance?"}
                    ]
                },
                "expected_response": "Here are several optimization strategies..."
            }
        ]

        # Execute each scenario
        for scenario in test_scenarios:
            # Mock agent response
            mock_mcp_client.send_message.return_value = create_agent_response(
                scenario["expected_response"]
            )

            # Send request
            response = await async_client.post(
                "/v1/chat/completions",
                json=scenario["request"],
                headers=auth_headers
            )

            # Verify response
            assert response.status_code == 200, f"Failed at scenario: {scenario['name']}"

            data = response.json()
            assert data["model"] == "cursor-agent-local"
            assert len(data["choices"]) == 1
            assert "assistant" in data["choices"][0]["message"]["role"]

            # Check performance
            assert "usage" in data
            assert data["usage"]["total_tokens"] > 0

    @pytest.mark.asyncio
    async def test_streaming_conversation_flow(self, async_client, auth_headers, mock_mcp_client):
        """Test conversation flow dengan streaming responses"""

        # Mock streaming conversation
        conversation_history = []

        async def mock_streaming_response(request_id, response_text):
            words = response_text.split()
            for i, word in enumerate(words):
                yield {
                    "type": "model_stream_chunk",
                    "id": request_id,
                    "chunk": {
                        "delta": {"content": word + " "},
                        "finish_reason": None
                    },
                    "finished": i == len(words) - 1
                }

        # Conversation steps
        conversation_steps = [
            {
                "user": "Hello, I need help with Python",
                "assistant": "Hello! I'd be happy to help you with Python programming."
            },
            {
                "user": "How do I read a file?",
                "assistant": "You can read a file in Python using the open() function with a context manager."
            },
            {
                "user": "Can you show me an example?",
                "assistant": "Sure! Here's an example: with open('file.txt', 'r') as f: content = f.read()"
            }
        ]

        for step in conversation_steps:
            # Add user message ke conversation
            conversation_history.append({"role": "user", "content": step["user"]})

            # Mock streaming response
            mock_mcp_client.stream_responses.return_value = mock_streaming_response(
                "test-id", step["assistant"]
            )

            # Send streaming request
            request = {
                "model": "cursor-agent-local",
                "messages": conversation_history.copy(),
                "stream": True
            }

            response = await async_client.post(
                "/v1/chat/completions",
                json=request,
                headers=auth_headers
            )

            assert response.status_code == 200
            assert response.headers["content-type"] == "text/event-stream; charset=utf-8"

            # Collect streaming response
            collected_content = ""
            async for line in response.aiter_lines():
                if line.startswith("data: ") and not line.endswith("[DONE]"):
                    try:
                        chunk_data = json.loads(line[6:])
                        delta = chunk_data["choices"][0]["delta"]
                        if "content" in delta:
                            collected_content += delta["content"]
                    except json.JSONDecodeError:
                        pass

            # Add assistant response ke conversation
            conversation_history.append({"role": "assistant", "content": collected_content.strip()})

            # Verify content
            assert step["assistant"] in collected_content

    @pytest.mark.asyncio
    async def test_error_recovery_flow(self, async_client, auth_headers, mock_mcp_client):
        """Test error handling dan recovery"""

        # Test scenario 1: MCP connection error
        mock_mcp_client.send_message.side_effect = ConnectionError("MCP bridge unavailable")

        request = {
            "model": "cursor-agent-local",
            "messages": [{"role": "user", "content": "Test message"}]
        }

        response = await async_client.post(
            "/v1/chat/completions",
            json=request,
            headers=auth_headers
        )

        assert response.status_code == 500  # Internal server error

        # Test scenario 2: Recovery setelah connection restored
        mock_mcp_client.send_message.side_effect = None
        mock_mcp_client.send_message.return_value = {
            "type": "model_result",
            "id": "test-id",
            "choices": [
                {
                    "index": 0,
                    "message": {"role": "assistant", "content": "Connection restored!"},
                    "finish_reason": "stop"
                }
            ]
        }

        response = await async_client.post(
            "/v1/chat/completions",
            json=request,
            headers=auth_headers
        )

        assert response.status_code == 200
        data = response.json()
        assert "Connection restored!" in data["choices"][0]["message"]["content"]
```

---

## 📋 Testing Checklist

### Pre-Test Setup

- [ ] Test environment configured
- [ ] Test configuration loaded
- [ ] Mock services prepared
- [ ] Test data generated
- [ ] Dependencies installed

### Unit Testing

- [ ] **Security Manager**

  - [ ] API key verification (valid/invalid)
  - [ ] Session ID generation (uniqueness)
  - [ ] Request data validation
  - [ ] Content sanitization (XSS prevention)
  - [ ] Timing attack resistance

- [ ] **Session Manager**

  - [ ] Session creation
  - [ ] Session retrieval (valid/invalid)
  - [ ] Session expiration
  - [ ] Cleanup expired sessions
  - [ ] Maximum sessions limit

- [ ] **Rate Limiter**

  - [ ] Request rate limiting
  - [ ] Token rate limiting
  - [ ] Different clients separation
  - [ ] Window reset behavior
  - [ ] Cleanup old data

- [ ] **MCP Client**
  - [ ] Connection establishment
  - [ ] Message sending/receiving
  - [ ] Streaming responses
  - [ ] Connection recovery
  - [ ] Error handling

### Integration Testing

- [ ] **OpenAI API**

  - [ ] Chat completions (non-streaming)
  - [ ] Chat completions (streaming)
  - [ ] Models list
  - [ ] Authentication
  - [ ] Error responses

- [ ] **Ollama API**

  - [ ] Chat endpoint
  - [ ] Tags endpoint
  - [ ] Streaming responses
  - [ ] Format conversion

- [ ] **Management API**
  - [ ] Health check
  - [ ] Metrics endpoint
  - [ ] Configuration endpoint

### Security Testing

- [ ] **Authentication**

  - [ ] Invalid API key rejection
  - [ ] Timing attack resistance
  - [ ] Session security

- [ ] **Input Validation**

  - [ ] Model name validation
  - [ ] Message format validation
  - [ ] Parameter validation
  - [ ] Content sanitization

- [ ] **Rate Limiting**
  - [ ] Request limits enforced
  - [ ] Token limits enforced
  - [ ] Client separation
  - [ ] Window behavior

### Performance Testing

- [ ] **Load Testing**

  - [ ] Concurrent requests
  - [ ] Streaming performance
  - [ ] Memory usage under load
  - [ ] Response time requirements

- [ ] **Scalability Testing**
  - [ ] Maximum sessions
  - [ ] Maximum concurrent connections
  - [ ] Resource cleanup

### End-to-End Testing

- [ ] **Complete Workflows**

  - [ ] Development workflow (explain → write → debug → optimize)
  - [ ] Streaming conversation flow
  - [ ] Error recovery flow
  - [ ] Multi-client scenarios

- [ ] **Error Scenarios**
  - [ ] MCP connection failures
  - [ ] Invalid requests
  - [ ] Rate limit exceeded
  - [ ] Service unavailable

### Test Execution

```bash
# Run semua tests
pytest

# Run specific test categories
pytest tests/unit/
pytest tests/integration/
pytest tests/security/
pytest tests/performance/
pytest tests/e2e/

# Run dengan coverage
pytest --cov=local_provider --cov-report=html

# Run performance tests
pytest tests/performance/ -v

# Run security tests
pytest tests/security/ -v

# Run dengan specific markers
pytest -m "not slow"  # Skip slow tests
pytest -m "security"  # Run only security tests
```

### Test Reporting

```bash
# Generate coverage report
coverage html

# Generate performance report
pytest tests/performance/ --benchmark-json=benchmark.json

# Generate security test report
pytest tests/security/ --html=security_report.html

# Generate comprehensive test report
pytest --html=test_report.html --self-contained-html
```

---

**Selesai**: Dokumentasi lengkap untuk MCP Local Provider sudah selesai dibuat dalam bahasa Indonesia
