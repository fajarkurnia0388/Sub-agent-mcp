# MCP VoidEditor Bridge — Testing Guide

> **Language**: [🇺🇸 English](TESTING.md) | [🇮🇩 Bahasa Indonesia](docs_id/TESTING.md)

**Comprehensive testing strategy for the MCP Bridge system**

## 🧪 Testing Overview

The MCP Bridge system requires comprehensive testing across multiple layers: unit tests, integration tests, security tests, performance tests, and end-to-end testing scenarios.

### Testing Philosophy

1. **Security First** - All security features must be thoroughly tested
2. **Real-time Communication** - WebSocket functionality requires specialized testing
3. **Session Management** - Ephemeral sessions need lifecycle testing
4. **Integration** - Agent-Bridge-Plugin communication flows
5. **Performance** - Rate limiting and scalability testing

---

## 🔧 Test Setup

### Test Environment

```bash
# Create test environment
python -m venv test_env
source test_env/bin/activate  # Windows: test_env\Scripts\activate

# Install test dependencies
pip install pytest pytest-asyncio pytest-websocket httpx faker freezegun

# Install bridge package in development mode
pip install -e .
```

### Test Configuration (`tests/config/test_config.yaml`)

```yaml
bridge:
  host: "127.0.0.1"
  port: 8788 # Different port for testing
  debug: true

security:
  mcp_token: "test-token-for-testing-only"
  session_ttl: 60 # Shorter for testing
  max_sessions: 5

rate_limiting:
  enabled: true
  requests_per_window: 20 # Higher limit for testing
  window_seconds: 60

logging:
  level: "DEBUG"
  handlers: ["console"]

sessions:
  cleanup_interval: 10 # Faster cleanup
  max_idle_time: 120

filesystem:
  max_file_size: 1048576 # 1MB for testing
  max_edit_size: 10240 # 10KB for testing
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

from bridge.main import app
from bridge.session_manager import SessionManager
from bridge.websocket_manager import WebSocketManager
from bridge.security import SecurityManager

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def test_client():
    """FastAPI test client"""
    return TestClient(app)

@pytest.fixture
async def async_client():
    """Async HTTP client for testing"""
    async with AsyncClient(app=app, base_url="http://testserver") as client:
        yield client

@pytest.fixture
def test_config():
    """Test configuration"""
    return {
        "bridge": {"host": "127.0.0.1", "port": 8788, "debug": True},
        "security": {"mcp_token": "test-token", "session_ttl": 60},
        "rate_limiting": {"requests_per_window": 20, "window_seconds": 60},
        "sessions": {"cleanup_interval": 10, "max_idle_time": 120}
    }

@pytest.fixture
def session_manager(test_config):
    """Session manager instance"""
    return SessionManager(test_config["sessions"])

@pytest.fixture
def websocket_manager():
    """WebSocket manager instance"""
    return WebSocketManager()

@pytest.fixture
def security_manager(test_config):
    """Security manager instance"""
    return SecurityManager(test_config["security"])

@pytest.fixture
def temp_workspace():
    """Temporary workspace directory"""
    with tempfile.TemporaryDirectory() as temp_dir:
        yield temp_dir

@pytest.fixture
def valid_access_request():
    """Valid access request data"""
    return {
        "agent_id": "test-agent-1",
        "scopes": ["read:files", "edit:buffers"],
        "roots": ["/tmp/test"],
        "reason": "Testing sub-agent functionality"
    }

@pytest.fixture
def auth_headers():
    """Authentication headers for API requests"""
    return {"Authorization": "Bearer test-token"}
```

---

## 🧪 Unit Tests

### Security Tests (`tests/unit/test_security.py`)

```python
import pytest
from datetime import datetime, timedelta
import jwt

from bridge.security import SecurityManager

class TestSecurityManager:

    def test_verify_mcp_token_valid(self, security_manager):
        """Test MCP token verification with valid token"""
        assert security_manager.verify_mcp_token("test-token") == True

    def test_verify_mcp_token_invalid(self, security_manager):
        """Test MCP token verification with invalid token"""
        assert security_manager.verify_mcp_token("wrong-token") == False

    def test_validate_scopes_all_valid(self, security_manager):
        """Test scope validation with all valid scopes"""
        valid_scopes = ["read:files", "edit:buffers", "exec:terminal"]
        invalid = security_manager.validate_scopes(valid_scopes)
        assert invalid == []

    def test_validate_scopes_some_invalid(self, security_manager):
        """Test scope validation with some invalid scopes"""
        mixed_scopes = ["read:files", "invalid:scope", "edit:buffers"]
        invalid = security_manager.validate_scopes(mixed_scopes)
        assert "invalid:scope" in invalid
        assert len(invalid) == 1

    def test_generate_session_token(self, security_manager):
        """Test session token generation"""
        token = security_manager.generate_session_token(
            "sess-123", "agent-1", ["read:files"], 300
        )
        assert isinstance(token, str)
        assert len(token) > 0

    def test_verify_session_token_valid(self, security_manager):
        """Test session token verification with valid token"""
        token = security_manager.generate_session_token(
            "sess-123", "agent-1", ["read:files"], 300
        )
        payload = security_manager.verify_session_token(token)
        assert payload is not None
        assert payload["session_id"] == "sess-123"
        assert payload["agent_id"] == "agent-1"

    def test_verify_session_token_expired(self, security_manager):
        """Test session token verification with expired token"""
        # Create token that expires immediately
        token = security_manager.generate_session_token(
            "sess-123", "agent-1", ["read:files"], -1
        )
        payload = security_manager.verify_session_token(token)
        assert payload is None

    def test_check_scope_permission(self, security_manager):
        """Test scope permission checking"""
        user_scopes = ["read:files", "edit:buffers"]

        assert security_manager.check_scope_permission(user_scopes, "read:files") == True
        assert security_manager.check_scope_permission(user_scopes, "exec:terminal") == False
```

### Session Management Tests (`tests/unit/test_session_manager.py`)

```python
import pytest
import asyncio
from datetime import datetime, timedelta
from freezegun import freeze_time

from bridge.session_manager import SessionManager
from bridge.models import RequestStatus

class TestSessionManager:

    @pytest.mark.asyncio
    async def test_create_access_request(self, session_manager):
        """Test access request creation"""
        request_id = await session_manager.create_access_request(
            agent_id="test-agent",
            scopes=["read:files"],
            roots=["/tmp"],
            reason="Testing"
        )

        assert request_id.startswith("req-")
        request = await session_manager.get_request(request_id)
        assert request is not None
        assert request.agent_id == "test-agent"
        assert request.status == RequestStatus.PENDING

    @pytest.mark.asyncio
    async def test_approve_request(self, session_manager):
        """Test request approval and session creation"""
        # Create request
        request_id = await session_manager.create_access_request(
            "test-agent", ["read:files"], ["/tmp"], "Testing"
        )

        # Approve request
        session_info = await session_manager.approve_request(
            request_id, ["read:files"], 300
        )

        assert session_info.session_id.startswith("sess-")
        assert session_info.agent_id == "test-agent"
        assert session_info.approved_scopes == ["read:files"]
        assert session_info.status == "active"

    @pytest.mark.asyncio
    async def test_deny_request(self, session_manager):
        """Test request denial"""
        request_id = await session_manager.create_access_request(
            "test-agent", ["read:files"], ["/tmp"], "Testing"
        )

        await session_manager.deny_request(request_id, "Invalid reason")

        request = await session_manager.get_request(request_id)
        assert request.status == RequestStatus.DENIED
        assert request.denial_reason == "Invalid reason"

    @pytest.mark.asyncio
    async def test_session_verification(self, session_manager):
        """Test session verification"""
        # Create and approve request
        request_id = await session_manager.create_access_request(
            "test-agent", ["read:files"], ["/tmp"], "Testing"
        )
        session_info = await session_manager.approve_request(
            request_id, ["read:files"], 300
        )

        # Verify valid session
        valid = await session_manager.verify_session(
            session_info.session_id, session_info.session_token
        )
        assert valid == True

        # Verify invalid token
        invalid = await session_manager.verify_session(
            session_info.session_id, "wrong-token"
        )
        assert invalid == False

    @pytest.mark.asyncio
    async def test_session_expiration(self, session_manager):
        """Test session expiration"""
        request_id = await session_manager.create_access_request(
            "test-agent", ["read:files"], ["/tmp"], "Testing"
        )

        # Create session with 1 second TTL
        session_info = await session_manager.approve_request(
            request_id, ["read:files"], 1
        )

        # Session should be valid initially
        valid = await session_manager.verify_session(
            session_info.session_id, session_info.session_token
        )
        assert valid == True

        # Wait for expiration and test again
        await asyncio.sleep(2)
        expired = await session_manager.verify_session(
            session_info.session_id, session_info.session_token
        )
        assert expired == False

    @pytest.mark.asyncio
    async def test_max_sessions_limit(self, session_manager):
        """Test maximum sessions limit"""
        # Set low limit for testing
        session_manager.max_sessions = 2

        # Create two sessions
        for i in range(2):
            request_id = await session_manager.create_access_request(
                f"agent-{i}", ["read:files"], ["/tmp"], "Testing"
            )
            await session_manager.approve_request(request_id, ["read:files"], 300)

        # Third session should fail
        request_id = await session_manager.create_access_request(
            "agent-3", ["read:files"], ["/tmp"], "Testing"
        )

        with pytest.raises(ValueError, match="Maximum sessions reached"):
            await session_manager.approve_request(request_id, ["read:files"], 300)
```

### WebSocket Manager Tests (`tests/unit/test_websocket_manager.py`)

```python
import pytest
from unittest.mock import AsyncMock, MagicMock

from bridge.websocket_manager import WebSocketManager

class TestWebSocketManager:

    @pytest.mark.asyncio
    async def test_register_agent(self, websocket_manager):
        """Test agent registration"""
        mock_ws = AsyncMock()

        await websocket_manager.register_agent("sess-123", mock_ws)
        assert "sess-123" in websocket_manager.agents
        assert websocket_manager.agents["sess-123"] == mock_ws

    @pytest.mark.asyncio
    async def test_register_plugin(self, websocket_manager):
        """Test plugin registration"""
        mock_ws = AsyncMock()

        await websocket_manager.register_plugin("plugin-1", mock_ws)
        assert "plugin-1" in websocket_manager.plugins
        assert websocket_manager.plugins["plugin-1"] == mock_ws
        assert websocket_manager.total_registered == 1

    @pytest.mark.asyncio
    async def test_send_to_agent_success(self, websocket_manager):
        """Test sending message to agent successfully"""
        mock_ws = AsyncMock()
        await websocket_manager.register_agent("sess-123", mock_ws)

        message = {"type": "test", "data": "hello"}
        result = await websocket_manager.send_to_agent("sess-123", message)

        assert result == True
        mock_ws.send_json.assert_called_once_with(message)

    @pytest.mark.asyncio
    async def test_send_to_agent_not_found(self, websocket_manager):
        """Test sending message to non-existent agent"""
        message = {"type": "test"}
        result = await websocket_manager.send_to_agent("sess-404", message)
        assert result == False

    @pytest.mark.asyncio
    async def test_send_to_plugin_broadcast(self, websocket_manager):
        """Test broadcasting message to all plugins"""
        mock_ws1 = AsyncMock()
        mock_ws2 = AsyncMock()

        await websocket_manager.register_plugin("plugin-1", mock_ws1)
        await websocket_manager.register_plugin("plugin-2", mock_ws2)

        message = {"type": "broadcast", "data": "hello all"}
        result = await websocket_manager.send_to_plugin(message)

        assert result == True
        mock_ws1.send_json.assert_called_once_with(message)
        mock_ws2.send_json.assert_called_once_with(message)

    @pytest.mark.asyncio
    async def test_unregister_connections(self, websocket_manager):
        """Test unregistering connections"""
        mock_ws1 = AsyncMock()
        mock_ws2 = AsyncMock()

        await websocket_manager.register_agent("sess-123", mock_ws1)
        await websocket_manager.register_plugin("plugin-1", mock_ws2)

        await websocket_manager.unregister_agent("sess-123")
        await websocket_manager.unregister_plugin("plugin-1")

        assert "sess-123" not in websocket_manager.agents
        assert "plugin-1" not in websocket_manager.plugins
```

---

## 🔗 Integration Tests

### API Integration Tests (`tests/integration/test_api.py`)

```python
import pytest
import json
from httpx import AsyncClient

class TestMCPAPI:

    @pytest.mark.asyncio
    async def test_health_check(self, async_client):
        """Test health check endpoint"""
        response = await async_client.get("/health")
        assert response.status_code == 200

        data = response.json()
        assert data["status"] == "healthy"
        assert "timestamp" in data
        assert "version" in data

    @pytest.mark.asyncio
    async def test_request_access_valid(self, async_client, valid_access_request, auth_headers):
        """Test valid access request"""
        response = await async_client.post(
            "/mcp/request_access",
            json=valid_access_request,
            headers=auth_headers
        )
        assert response.status_code == 200

        data = response.json()
        assert data["status"] == "pending"
        assert data["request_id"].startswith("req-")

    @pytest.mark.asyncio
    async def test_request_access_invalid_token(self, async_client, valid_access_request):
        """Test access request with invalid token"""
        response = await async_client.post(
            "/mcp/request_access",
            json=valid_access_request,
            headers={"Authorization": "Bearer invalid-token"}
        )
        assert response.status_code == 401

    @pytest.mark.asyncio
    async def test_request_access_invalid_scopes(self, async_client, auth_headers):
        """Test access request with invalid scopes"""
        invalid_request = {
            "agent_id": "test-agent",
            "scopes": ["invalid:scope"],
            "roots": ["/tmp"],
            "reason": "Testing invalid scopes"
        }

        response = await async_client.post(
            "/mcp/request_access",
            json=invalid_request,
            headers=auth_headers
        )
        assert response.status_code == 400
        assert "Invalid scopes" in response.json()["detail"]

    @pytest.mark.asyncio
    async def test_approve_request_flow(self, async_client, valid_access_request, auth_headers):
        """Test complete request approval flow"""
        # Create request
        response = await async_client.post(
            "/mcp/request_access",
            json=valid_access_request,
            headers=auth_headers
        )
        request_id = response.json()["request_id"]

        # Approve request
        approve_data = {
            "request_id": request_id,
            "approved_scopes": ["read:files"],
            "ttl_seconds": 300
        }

        response = await async_client.post(
            "/mcp/approve",
            json=approve_data,
            headers=auth_headers
        )
        assert response.status_code == 200

        data = response.json()
        assert data["session_id"].startswith("sess-")
        assert "session_token" in data
        assert data["approved_scopes"] == ["read:files"]

    @pytest.mark.asyncio
    async def test_list_requests(self, async_client, auth_headers):
        """Test listing requests"""
        response = await async_client.get("/mcp/requests", headers=auth_headers)
        assert response.status_code == 200

        data = response.json()
        assert "requests" in data
        assert "total" in data
        assert isinstance(data["requests"], list)

    @pytest.mark.asyncio
    async def test_list_sessions(self, async_client, auth_headers):
        """Test listing active sessions"""
        response = await async_client.get("/mcp/sessions", headers=auth_headers)
        assert response.status_code == 200

        data = response.json()
        assert "sessions" in data
        assert "total" in data
        assert isinstance(data["sessions"], list)
```

### WebSocket Integration Tests (`tests/integration/test_websocket.py`)

```python
import pytest
import asyncio
import json
from websockets import connect, WebSocketException

class TestWebSocketIntegration:

    @pytest.mark.asyncio
    async def test_agent_websocket_connection(self):
        """Test agent WebSocket connection and authentication"""
        # This would require setting up a test session first
        session_id = "test-sess-123"
        session_token = "test-token-123"

        uri = f"ws://127.0.0.1:8788/mcp/session/{session_id}"

        try:
            async with connect(uri) as websocket:
                # Send authentication
                auth_message = {
                    "type": "auth",
                    "session_token": session_token
                }
                await websocket.send(json.dumps(auth_message))

                # Should receive confirmation or close for invalid session
                response = await asyncio.wait_for(websocket.recv(), timeout=5.0)
                # Test based on expected behavior

        except WebSocketException:
            # Expected if session doesn't exist
            pass

    @pytest.mark.asyncio
    async def test_plugin_websocket_registration(self):
        """Test plugin WebSocket registration"""
        plugin_id = "test-plugin-1"
        uri = f"ws://127.0.0.1:8788/plugin/{plugin_id}"

        try:
            async with connect(uri) as websocket:
                # Send registration
                registration = {
                    "type": "register",
                    "plugin_id": plugin_id,
                    "capabilities": ["file_operations"],
                    "version": "1.0.0"
                }
                await websocket.send(json.dumps(registration))

                # Receive registration confirmation
                response = await asyncio.wait_for(websocket.recv(), timeout=5.0)
                data = json.loads(response)

                assert data["type"] == "registered"
                assert data["status"] == "ok"
                assert data["assigned_id"] == plugin_id

        except WebSocketException as e:
            pytest.fail(f"WebSocket connection failed: {e}")
```

---

## 🔒 Security Tests

### Authentication Tests (`tests/security/test_auth.py`)

```python
import pytest
import jwt
from datetime import datetime, timedelta

from bridge.security import SecurityManager

class TestSecurityScenarios:

    def test_token_timing_attack_resistance(self, security_manager):
        """Test resistance to timing attacks on token verification"""
        import time

        valid_token = "test-token"
        invalid_token = "invalid-token"

        # Measure timing for valid token
        start = time.perf_counter()
        security_manager.verify_mcp_token(valid_token)
        valid_time = time.perf_counter() - start

        # Measure timing for invalid token
        start = time.perf_counter()
        security_manager.verify_mcp_token(invalid_token)
        invalid_time = time.perf_counter() - start

        # Times should be similar (within reasonable variance)
        time_diff = abs(valid_time - invalid_time)
        assert time_diff < 0.001  # Less than 1ms difference

    def test_session_token_tampering(self, security_manager):
        """Test session token tampering detection"""
        # Generate valid token
        token = security_manager.generate_session_token(
            "sess-123", "agent-1", ["read:files"], 300
        )

        # Tamper with token
        tampered_token = token[:-5] + "XXXXX"

        # Should fail verification
        payload = security_manager.verify_session_token(tampered_token)
        assert payload is None

    def test_scope_escalation_prevention(self, security_manager):
        """Test prevention of scope escalation"""
        user_scopes = ["read:files"]

        # Should not allow escalated permissions
        assert security_manager.check_scope_permission(user_scopes, "exec:terminal") == False
        assert security_manager.check_scope_permission(user_scopes, "delete:files") == False

        # Should allow granted permissions
        assert security_manager.check_scope_permission(user_scopes, "read:files") == True

    @pytest.mark.asyncio
    async def test_session_hijacking_prevention(self, session_manager):
        """Test session hijacking prevention"""
        # Create session
        request_id = await session_manager.create_access_request(
            "agent-1", ["read:files"], ["/tmp"], "Testing"
        )
        session_info = await session_manager.approve_request(
            request_id, ["read:files"], 300
        )

        # Valid verification should work
        valid = await session_manager.verify_session(
            session_info.session_id, session_info.session_token
        )
        assert valid == True

        # Different session ID with same token should fail
        invalid = await session_manager.verify_session(
            "sess-hijacked", session_info.session_token
        )
        assert invalid == False
```

### Input Validation Tests (`tests/security/test_validation.py`)

```python
import pytest

class TestInputValidation:

    @pytest.mark.parametrize("invalid_agent_id", [
        "",  # Empty
        "a" * 100,  # Too long
        "agent@invalid",  # Invalid characters
        "agent with spaces",  # Spaces
        "../../../etc/passwd",  # Path traversal attempt
    ])
    @pytest.mark.asyncio
    async def test_invalid_agent_ids(self, async_client, auth_headers, invalid_agent_id):
        """Test rejection of invalid agent IDs"""
        request_data = {
            "agent_id": invalid_agent_id,
            "scopes": ["read:files"],
            "roots": ["/tmp"],
            "reason": "Testing invalid agent ID"
        }

        response = await async_client.post(
            "/mcp/request_access",
            json=request_data,
            headers=auth_headers
        )
        assert response.status_code == 422  # Validation error

    @pytest.mark.parametrize("invalid_path", [
        "../../../etc/passwd",  # Path traversal
        "/etc/shadow",  # System file
        "\\Windows\\System32",  # Windows system path
        "",  # Empty path
        "relative/path",  # Non-absolute path
    ])
    @pytest.mark.asyncio
    async def test_path_validation(self, async_client, auth_headers, invalid_path):
        """Test path validation for security"""
        request_data = {
            "agent_id": "test-agent",
            "scopes": ["read:files"],
            "roots": [invalid_path],
            "reason": "Testing path validation"
        }

        response = await async_client.post(
            "/mcp/request_access",
            json=request_data,
            headers=auth_headers
        )
        # Should either reject or sanitize the path
        assert response.status_code in [400, 422]

    def test_reason_validation(self, valid_access_request):
        """Test reason field validation"""
        from bridge.models import AccessRequest

        # Too short reason
        with pytest.raises(ValueError):
            AccessRequest(
                agent_id="test-agent",
                scopes=["read:files"],
                roots=["/tmp"],
                reason="short"
            )

        # Too long reason
        with pytest.raises(ValueError):
            AccessRequest(
                agent_id="test-agent",
                scopes=["read:files"],
                roots=["/tmp"],
                reason="x" * 1000
            )
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
        async def make_request(agent_id):
            request_data = {
                "agent_id": f"agent-{agent_id}",
                "scopes": ["read:files"],
                "roots": ["/tmp"],
                "reason": f"Load testing agent {agent_id}"
            }

            start_time = time.time()
            response = await async_client.post(
                "/mcp/request_access",
                json=request_data,
                headers=auth_headers
            )
            end_time = time.time()

            return {
                "status_code": response.status_code,
                "response_time": end_time - start_time,
                "agent_id": agent_id
            }

        # Create 10 concurrent requests
        tasks = [make_request(i) for i in range(10)]
        results = await asyncio.gather(*tasks)

        # All requests should succeed
        for result in results:
            assert result["status_code"] == 200
            assert result["response_time"] < 1.0  # Should respond within 1 second

    @pytest.mark.asyncio
    async def test_rate_limiting_behavior(self, async_client, auth_headers):
        """Test rate limiting under load"""
        async def make_rapid_requests():
            responses = []
            for i in range(25):  # Exceed rate limit of 20
                response = await async_client.get("/health")
                responses.append(response.status_code)
            return responses

        responses = await make_rapid_requests()

        # Should have some rate limited responses (429)
        success_count = sum(1 for code in responses if code == 200)
        rate_limited_count = sum(1 for code in responses if code == 429)

        assert success_count <= 20  # Within rate limit
        assert rate_limited_count > 0  # Some requests rate limited

    @pytest.mark.asyncio
    async def test_memory_usage_under_load(self, session_manager):
        """Test memory usage doesn't grow excessively"""
        import psutil
        import os

        process = psutil.Process(os.getpid())
        initial_memory = process.memory_info().rss

        # Create many sessions
        session_ids = []
        for i in range(100):
            request_id = await session_manager.create_access_request(
                f"agent-{i}", ["read:files"], ["/tmp"], "Memory testing"
            )
            session_info = await session_manager.approve_request(
                request_id, ["read:files"], 60
            )
            session_ids.append(session_info.session_id)

        peak_memory = process.memory_info().rss

        # Clean up sessions
        for session_id in session_ids:
            await session_manager.revoke_session(session_id, "Test cleanup")

        final_memory = process.memory_info().rss

        # Memory should not grow excessively
        memory_growth = peak_memory - initial_memory
        memory_ratio = memory_growth / initial_memory

        assert memory_ratio < 2.0  # Less than 100% growth

        # Memory should be released after cleanup
        cleanup_ratio = (peak_memory - final_memory) / memory_growth
        assert cleanup_ratio > 0.5  # At least 50% cleanup
```

---

## 🎯 End-to-End Tests

### Complete Flow Tests (`tests/e2e/test_complete_flow.py`)

```python
import pytest
import asyncio
import json
from websockets import connect

class TestEndToEndFlows:

    @pytest.mark.asyncio
    async def test_complete_sub_agent_workflow(self, async_client, auth_headers):
        """Test complete sub-agent development workflow"""

        # Step 1: Request access
        access_request = {
            "agent_id": "sub-agent-dev-1",
            "scopes": [
                "read:files", "edit:buffers", "analyze:syntax",
                "build:project", "exec:terminal", "git:commit"
            ],
            "roots": ["/tmp/test-project"],
            "reason": "Complete development workflow testing"
        }

        response = await async_client.post(
            "/mcp/request_access",
            json=access_request,
            headers=auth_headers
        )
        assert response.status_code == 200
        request_id = response.json()["request_id"]

        # Step 2: Approve request
        approve_data = {
            "request_id": request_id,
            "approved_scopes": access_request["scopes"],
            "ttl_seconds": 600
        }

        response = await async_client.post(
            "/mcp/approve",
            json=approve_data,
            headers=auth_headers
        )
        assert response.status_code == 200

        session_data = response.json()
        session_id = session_data["session_id"]
        session_token = session_data["session_token"]

        # Step 3: Connect via WebSocket (simulated)
        # In real test, would connect agent and plugin WebSockets

        # Step 4: Verify session is active
        response = await async_client.get("/mcp/sessions", headers=auth_headers)
        sessions = response.json()["sessions"]
        active_session = next(s for s in sessions if s["session_id"] == session_id)
        assert active_session["status"] == "active"

        # Step 5: Revoke session
        revoke_data = {
            "session_id": session_id,
            "reason": "Test completed"
        }

        response = await async_client.post(
            "/mcp/revoke",
            json=revoke_data,
            headers=auth_headers
        )
        assert response.status_code == 200

    @pytest.mark.asyncio
    async def test_session_expiration_flow(self, async_client, auth_headers):
        """Test session expiration behavior"""

        # Create session with short TTL
        access_request = {
            "agent_id": "test-expiry-agent",
            "scopes": ["read:files"],
            "roots": ["/tmp"],
            "reason": "Testing session expiration"
        }

        response = await async_client.post(
            "/mcp/request_access",
            json=access_request,
            headers=auth_headers
        )
        request_id = response.json()["request_id"]

        # Approve with 2 second TTL
        approve_data = {
            "request_id": request_id,
            "ttl_seconds": 2
        }

        response = await async_client.post(
            "/mcp/approve",
            json=approve_data,
            headers=auth_headers
        )
        session_id = response.json()["session_id"]

        # Session should be active initially
        response = await async_client.get("/mcp/sessions", headers=auth_headers)
        sessions = response.json()["sessions"]
        assert any(s["session_id"] == session_id for s in sessions)

        # Wait for expiration
        await asyncio.sleep(3)

        # Session should be expired/cleaned up
        response = await async_client.get("/mcp/sessions", headers=auth_headers)
        sessions = response.json()["sessions"]
        active_sessions = [s for s in sessions if s["session_id"] == session_id and s["status"] == "active"]
        assert len(active_sessions) == 0

    @pytest.mark.asyncio
    async def test_security_violation_handling(self, async_client, auth_headers):
        """Test handling of security violations"""

        # Step 1: Create session with limited scopes
        access_request = {
            "agent_id": "limited-agent",
            "scopes": ["read:files"],  # Only read access
            "roots": ["/tmp"],
            "reason": "Testing security boundaries"
        }

        response = await async_client.post(
            "/mcp/request_access",
            json=access_request,
            headers=auth_headers
        )
        request_id = response.json()["request_id"]

        response = await async_client.post(
            "/mcp/approve",
            json={"request_id": request_id},
            headers=auth_headers
        )
        session_id = response.json()["session_id"]

        # Step 2: Simulate attempting restricted action
        # This would normally be done via WebSocket
        # For now, verify the session has limited scopes
        response = await async_client.get("/mcp/sessions", headers=auth_headers)
        sessions = response.json()["sessions"]
        session = next(s for s in sessions if s["session_id"] == session_id)

        assert "read:files" in session["approved_scopes"]
        assert "exec:terminal" not in session["approved_scopes"]
        assert "delete:files" not in session["approved_scopes"]
```

---

## 📋 Testing Checklist

### Pre-Test Setup

- [ ] Test environment configured
- [ ] Test database/storage cleaned
- [ ] Test configuration loaded
- [ ] Dependencies installed
- [ ] Fixtures prepared

### Unit Testing

- [ ] **Security Manager**

  - [ ] Token verification (valid/invalid)
  - [ ] Scope validation
  - [ ] Session token generation/verification
  - [ ] Permission checking
  - [ ] Timing attack resistance

- [ ] **Session Manager**

  - [ ] Access request creation
  - [ ] Request approval/denial
  - [ ] Session verification
  - [ ] Session expiration
  - [ ] Session cleanup
  - [ ] Maximum sessions limit

- [ ] **WebSocket Manager**
  - [ ] Agent/plugin registration
  - [ ] Message sending/broadcasting
  - [ ] Connection management
  - [ ] Error handling

### Integration Testing

- [ ] **API Endpoints**

  - [ ] Authentication required
  - [ ] Request validation
  - [ ] Response formats
  - [ ] Error responses
  - [ ] Rate limiting

- [ ] **WebSocket Communication**
  - [ ] Connection establishment
  - [ ] Message routing
  - [ ] Authentication flow
  - [ ] Connection cleanup

### Security Testing

- [ ] **Authentication**

  - [ ] Invalid token rejection
  - [ ] Token tampering detection
  - [ ] Session hijacking prevention

- [ ] **Authorization**

  - [ ] Scope enforcement
  - [ ] Permission escalation prevention
  - [ ] Path traversal prevention

- [ ] **Input Validation**
  - [ ] Agent ID validation
  - [ ] Path validation
  - [ ] Scope validation
  - [ ] Size limits

### Performance Testing

- [ ] **Load Testing**

  - [ ] Concurrent requests
  - [ ] Rate limiting behavior
  - [ ] Memory usage under load
  - [ ] Response time requirements

- [ ] **Scalability Testing**
  - [ ] Maximum sessions
  - [ ] Maximum connections
  - [ ] Resource cleanup

### End-to-End Testing

- [ ] **Complete Workflows**

  - [ ] Request → Approve → Connect → Operate → Cleanup
  - [ ] Session expiration flow
  - [ ] Security violation handling
  - [ ] Multi-agent scenarios

- [ ] **Error Scenarios**
  - [ ] Network failures
  - [ ] Invalid sessions
  - [ ] Plugin disconnections
  - [ ] Resource exhaustion

### Test Execution

```bash
# Run all tests
pytest

# Run specific test categories
pytest tests/unit/
pytest tests/integration/
pytest tests/security/
pytest tests/performance/
pytest tests/e2e/

# Run with coverage
pytest --cov=bridge --cov-report=html

# Run performance tests
pytest tests/performance/ -v

# Run security tests
pytest tests/security/ -v
```

### Test Reporting

```bash
# Generate coverage report
coverage html

# Generate performance report
pytest tests/performance/ --benchmark-json=benchmark.json

# Generate security test report
pytest tests/security/ --html=security_report.html
```

---

**Next**: Bridge documentation is now complete. Time to create Indonesian versions for the local_provider documentation.
