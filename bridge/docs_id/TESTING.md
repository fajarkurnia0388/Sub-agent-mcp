# MCP VoidEditor Bridge — Panduan Testing

> **Bahasa**: [🇺🇸 English](../TESTING.md) | [🇮🇩 Bahasa Indonesia](TESTING.md)

**Strategi testing komprehensif untuk sistem MCP Bridge**

## 🧪 Gambaran Testing

Sistem MCP Bridge memerlukan testing komprehensif di berbagai layer: unit tests, integration tests, security tests, performance tests, dan skenario end-to-end testing.

### Filosofi Testing

1. **Keamanan Utama** - Semua fitur keamanan harus diuji secara menyeluruh
2. **Komunikasi Real-time** - Fungsi WebSocket memerlukan testing khusus
3. **Manajemen Sesi** - Sesi efemeral memerlukan testing lifecycle
4. **Integrasi** - Alur komunikasi Agent-Bridge-Plugin
5. **Performance** - Rate limiting dan testing skalabilitas

---

## 🔧 Setup Testing

### Environment Testing

```bash
# Buat environment testing
python -m venv test_env
source test_env/bin/activate  # Windows: test_env\Scripts\activate

# Install dependencies testing
pip install pytest pytest-asyncio pytest-websocket httpx faker freezegun

# Install bridge package dalam mode development
pip install -e .
```

### Konfigurasi Testing (`tests/config/test_config.yaml`)

```yaml
bridge:
  host: "127.0.0.1"
  port: 8788 # Port berbeda untuk testing
  debug: true

security:
  mcp_token: "test-token-for-testing-only"
  session_ttl: 60 # Lebih pendek untuk testing
  max_sessions: 5

rate_limiting:
  requests_per_minute: 1000 # Lebih tinggi untuk testing
  burst_allowance: 100

filesystem:
  allowed_roots: ["/tmp/test_workspace"]
  max_file_size: 1048576 # 1MB untuk testing
  allowed_extensions: [".txt", ".js", ".py", ".md"]

logging:
  level: "DEBUG"
  file: "test_bridge.log"
  audit_file: "test_audit.log"
```

### Test Data Setup

```python
# tests/fixtures/test_data.py
import tempfile
import os
from pathlib import Path

class TestDataSetup:
    def __init__(self):
        self.temp_dir = None
        self.test_files = {}

    def setup_test_workspace(self):
        """Setup workspace testing"""
        self.temp_dir = tempfile.mkdtemp(prefix="mcp_bridge_test_")

        # Create test files
        test_files = {
            "test.txt": "Hello World",
            "test.js": "console.log('Hello World');",
            "test.py": "print('Hello World')",
            "test.md": "# Test Document\n\nThis is a test."
        }

        for filename, content in test_files.items():
            filepath = os.path.join(self.temp_dir, filename)
            with open(filepath, 'w') as f:
                f.write(content)
            self.test_files[filename] = filepath

        return self.temp_dir

    def cleanup(self):
        """Cleanup test data"""
        if self.temp_dir and os.path.exists(self.temp_dir):
            import shutil
            shutil.rmtree(self.temp_dir)
```

---

## 🔬 Unit Tests

### Security Manager Tests

```python
# tests/unit/test_security.py
import pytest
from bridge.security import SecurityManager
from bridge.models import AccessRequest

class TestSecurityManager:
    def setup_method(self):
        self.security_manager = SecurityManager()

    def test_verify_token_valid(self):
        """Test valid token verification"""
        valid_token = "test-token-for-testing-only"
        assert self.security_manager.verify_token(valid_token) == True

    def test_verify_token_invalid(self):
        """Test invalid token verification"""
        invalid_token = "invalid-token"
        assert self.security_manager.verify_token(invalid_token) == False

    def test_validate_scopes_valid(self):
        """Test valid scope validation"""
        valid_scopes = ["read:files", "edit:buffers"]
        assert self.security_manager.validate_scopes(valid_scopes) == True

    def test_validate_scopes_invalid(self):
        """Test invalid scope validation"""
        invalid_scopes = ["invalid:scope", "read:files"]
        assert self.security_manager.validate_scopes(invalid_scopes) == False

    def test_validate_root_path_allowed(self):
        """Test allowed root path validation"""
        allowed_path = "/tmp/test_workspace"
        assert self.security_manager.validate_root_path(allowed_path) == True

    def test_validate_root_path_blocked(self):
        """Test blocked root path validation"""
        blocked_path = "/etc/passwd"
        assert self.security_manager.validate_root_path(blocked_path) == False

    def test_generate_session_token(self):
        """Test session token generation"""
        token1 = self.security_manager.generate_session_token()
        token2 = self.security_manager.generate_session_token()

        assert len(token1) == 32
        assert len(token2) == 32
        assert token1 != token2  # Should be unique

    def test_validate_file_operation_allowed(self):
        """Test allowed file operation validation"""
        operation = "read"
        path = "/tmp/test_workspace/test.txt"

        assert self.security_manager.validate_file_operation(operation, path) == True

    def test_validate_file_operation_blocked_extension(self):
        """Test blocked file extension validation"""
        operation = "read"
        path = "/tmp/test_workspace/test.exe"

        assert self.security_manager.validate_file_operation(operation, path) == False

    def test_validate_file_operation_too_large(self):
        """Test file size validation"""
        operation = "write"
        path = "/tmp/test_workspace/test.txt"
        large_content = "x" * 10485761  # Larger than 10MB

        assert self.security_manager.validate_file_operation(operation, path, large_content) == False
```

### Session Manager Tests

```python
# tests/unit/test_session_manager.py
import pytest
import asyncio
from datetime import datetime, timedelta
from bridge.session_manager import SessionManager
from bridge.models import AccessRequest

class TestSessionManager:
    @pytest.fixture
    async def session_manager(self):
        manager = SessionManager()
        await manager.initialize()
        yield manager
        await manager.cleanup()

    @pytest.fixture
    def sample_request(self):
        return AccessRequest(
            agent_id="test-agent",
            scopes=["read:files", "edit:buffers"],
            root_path="/tmp/test_workspace",
            ttl=60
        )

    async def test_create_pending_session(self, session_manager, sample_request):
        """Test pending session creation"""
        session_id = await session_manager.create_pending_session(sample_request, "test-token")

        assert session_id is not None
        assert session_id in session_manager.pending_sessions

        session = session_manager.pending_sessions[session_id]
        assert session.agent_id == "test-agent"
        assert session.status == "pending"
        assert session.scopes == ["read:files", "edit:buffers"]

    async def test_create_active_session(self, session_manager, sample_request):
        """Test active session creation"""
        # Create pending session first
        session_id = await session_manager.create_pending_session(sample_request, "test-token")
        pending_session = session_manager.pending_sessions[session_id]

        # Create active session
        approved_scopes = ["read:files"]
        active_session = await session_manager.create_active_session(pending_session, approved_scopes)

        assert active_session.session_id == session_id
        assert active_session.status == "active"
        assert active_session.scopes == approved_scopes
        assert active_session.token is not None

        # Check pending session is removed
        assert session_id not in session_manager.pending_sessions
        assert session_id in session_manager.active_sessions

    async def test_get_session_by_token(self, session_manager, sample_request):
        """Test session retrieval by token"""
        session_id = await session_manager.create_pending_session(sample_request, "test-token")
        pending_session = session_manager.pending_sessions[session_id]

        active_session = await session_manager.create_active_session(pending_session, ["read:files"])

        retrieved_session = await session_manager.get_session_by_token(active_session.token)
        assert retrieved_session is not None
        assert retrieved_session.session_id == session_id

    async def test_update_activity(self, session_manager, sample_request):
        """Test session activity update"""
        session_id = await session_manager.create_pending_session(sample_request, "test-token")
        pending_session = session_manager.pending_sessions[session_id]

        active_session = await session_manager.create_active_session(pending_session, ["read:files"])

        original_activity = active_session.last_activity
        await asyncio.sleep(0.1)  # Small delay
        await session_manager.update_activity(session_id)

        updated_session = session_manager.active_sessions[session_id]
        assert updated_session.last_activity > original_activity

    async def test_revoke_session(self, session_manager, sample_request):
        """Test session revocation"""
        session_id = await session_manager.create_pending_session(sample_request, "test-token")
        pending_session = session_manager.pending_sessions[session_id]

        await session_manager.create_active_session(pending_session, ["read:files"])
        assert session_id in session_manager.active_sessions

        await session_manager.revoke_session(session_id)
        assert session_id not in session_manager.active_sessions
```

---

## 🔗 Integration Tests

### Bridge API Integration Tests

```python
# tests/integration/test_bridge_api.py
import pytest
import httpx
import asyncio
from fastapi.testclient import TestClient
from bridge.main import app

class TestBridgeAPI:
    @pytest.fixture
    def client(self):
        return TestClient(app)

    @pytest.fixture
    def auth_headers(self):
        return {"Authorization": "Bearer test-token-for-testing-only"}

    def test_health_check(self, client):
        """Test health check endpoint"""
        response = client.get("/health")
        assert response.status_code == 200

        data = response.json()
        assert "status" in data
        assert "timestamp" in data
        assert data["status"] == "healthy"

    def test_request_access_valid(self, client, auth_headers):
        """Test valid access request"""
        request_data = {
            "agent_id": "test-agent",
            "scopes": ["read:files", "edit:buffers"],
            "root_path": "/tmp/test_workspace",
            "ttl": 60,
            "reason": "Testing"
        }

        response = client.post("/mcp/request_access", json=request_data, headers=auth_headers)
        assert response.status_code == 202

        data = response.json()
        assert "request_id" in data
        assert data["status"] == "pending"

    def test_request_access_invalid_token(self, client):
        """Test access request with invalid token"""
        request_data = {
            "agent_id": "test-agent",
            "scopes": ["read:files"],
            "root_path": "/tmp/test_workspace"
        }

        response = client.post("/mcp/request_access", json=request_data)
        assert response.status_code == 401

    def test_request_access_invalid_scopes(self, client, auth_headers):
        """Test access request with invalid scopes"""
        request_data = {
            "agent_id": "test-agent",
            "scopes": ["invalid:scope"],
            "root_path": "/tmp/test_workspace"
        }

        response = client.post("/mcp/request_access", json=request_data, headers=auth_headers)
        assert response.status_code == 400

    def test_approve_access(self, client, auth_headers):
        """Test access approval"""
        # First create a request
        request_data = {
            "agent_id": "test-agent",
            "scopes": ["read:files", "edit:buffers"],
            "root_path": "/tmp/test_workspace"
        }

        response = client.post("/mcp/request_access", json=request_data, headers=auth_headers)
        request_id = response.json()["request_id"]

        # Then approve it
        approve_data = {
            "request_id": request_id,
            "approved_scopes": ["read:files"]
        }

        response = client.post("/mcp/approve", json=approve_data, headers=auth_headers)
        assert response.status_code == 200

        data = response.json()
        assert "session_id" in data
        assert "token" in data
        assert "expires_at" in data
        assert data["scopes"] == ["read:files"]
```

### WebSocket Integration Tests

```python
# tests/integration/test_websocket.py
import pytest
import asyncio
import json
from fastapi.testclient import TestClient
from bridge.main import app

class TestWebSocketIntegration:
    @pytest.fixture
    def client(self):
        return TestClient(app)

    async def test_websocket_authentication(self, client):
        """Test WebSocket authentication"""
        with client.websocket_connect("/ws") as websocket:
            # Send authentication
            auth_message = {
                "type": "auth",
                "token": "valid-session-token"
            }
            websocket.send_json(auth_message)

            # Should receive authentication response
            response = websocket.receive_json()
            assert response["type"] == "auth_success"

    async def test_websocket_command_flow(self, client):
        """Test WebSocket command flow"""
        with client.websocket_connect("/ws") as websocket:
            # Authenticate first
            auth_message = {"type": "auth", "token": "valid-session-token"}
            websocket.send_json(auth_message)
            websocket.receive_json()  # Auth response

            # Send command
            command_message = {
                "type": "cmd",
                "session_id": "test-session",
                "action": "file_operation",
                "params": {
                    "operation": "read",
                    "path": "/tmp/test_workspace/test.txt"
                }
            }
            websocket.send_json(command_message)

            # Should receive result
            response = websocket.receive_json()
            assert response["type"] == "result"
            assert response["status"] == "success"
```

---

## 🛡️ Security Tests

### Authentication Tests

```python
# tests/security/test_authentication.py
import pytest
from bridge.security import SecurityManager

class TestAuthentication:
    def test_token_verification_timing_attack(self):
        """Test token verification is not vulnerable to timing attacks"""
        security_manager = SecurityManager()

        # Test with tokens of different lengths
        short_token = "short"
        long_token = "very-long-token-that-should-take-longer"

        # Both should take similar time (within reasonable margin)
        import time

        start_time = time.time()
        security_manager.verify_token(short_token)
        short_time = time.time() - start_time

        start_time = time.time()
        security_manager.verify_token(long_token)
        long_time = time.time() - start_time

        # Difference should be minimal
        assert abs(short_time - long_time) < 0.01

    def test_session_token_uniqueness(self):
        """Test session tokens are unique"""
        security_manager = SecurityManager()

        tokens = set()
        for _ in range(1000):
            token = security_manager.generate_session_token()
            assert token not in tokens
            tokens.add(token)

    def test_path_traversal_protection(self):
        """Test protection against path traversal attacks"""
        security_manager = SecurityManager()

        malicious_paths = [
            "/tmp/test_workspace/../../../etc/passwd",
            "/tmp/test_workspace/..\\..\\..\\windows\\system32",
            "/tmp/test_workspace/....//....//....//etc/passwd"
        ]

        for path in malicious_paths:
            assert security_manager.validate_root_path(path) == False
```

### Authorization Tests

```python
# tests/security/test_authorization.py
import pytest
from bridge.security import SecurityManager

class TestAuthorization:
    def test_scope_isolation(self):
        """Test that scopes are properly isolated"""
        security_manager = SecurityManager()

        # Agent with limited scopes should not access restricted operations
        limited_scopes = ["read:files"]

        # Should be able to read files
        assert "read:files" in limited_scopes

        # Should not be able to write files
        assert "write:files" not in limited_scopes

        # Should not be able to execute terminal commands
        assert "exec:terminal" not in limited_scopes

    def test_root_path_isolation(self):
        """Test that root paths are properly isolated"""
        security_manager = SecurityManager()

        # Test that operations outside root path are blocked
        allowed_root = "/tmp/test_workspace"
        blocked_paths = [
            "/etc/passwd",
            "/home/user/private",
            "/var/log/system.log"
        ]

        for blocked_path in blocked_paths:
            assert security_manager.validate_root_path(blocked_path) == False
```

---

## ⚡ Performance Tests

### Load Testing

```python
# tests/performance/test_load.py
import pytest
import asyncio
import aiohttp
from concurrent.futures import ThreadPoolExecutor

class TestLoadPerformance:
    async def test_concurrent_sessions(self):
        """Test concurrent session creation"""
        async def create_session(session_id):
            async with aiohttp.ClientSession() as session:
                request_data = {
                    "agent_id": f"agent-{session_id}",
                    "scopes": ["read:files"],
                    "root_path": "/tmp/test_workspace"
                }
                headers = {"Authorization": "Bearer test-token-for-testing-only"}

                async with session.post(
                    "http://127.0.0.1:8788/mcp/request_access",
                    json=request_data,
                    headers=headers
                ) as response:
                    return await response.json()

        # Create 10 concurrent sessions
        tasks = [create_session(i) for i in range(10)]
        results = await asyncio.gather(*tasks)

        # All should succeed
        for result in results:
            assert result["status"] == "pending"

    def test_rate_limiting(self):
        """Test rate limiting functionality"""
        import requests

        # Send requests faster than rate limit
        for i in range(150):  # Exceed 100 requests per minute
            response = requests.post(
                "http://127.0.0.1:8788/mcp/request_access",
                json={
                    "agent_id": f"agent-{i}",
                    "scopes": ["read:files"],
                    "root_path": "/tmp/test_workspace"
                },
                headers={"Authorization": "Bearer test-token-for-testing-only"}
            )

            if i >= 100:  # Should be rate limited
                assert response.status_code == 429
```

---

## 🎯 End-to-End Tests

### Complete Workflow Tests

```python
# tests/e2e/test_complete_workflow.py
import pytest
import asyncio
from fastapi.testclient import TestClient
from bridge.main import app

class TestCompleteWorkflow:
    @pytest.fixture
    def client(self):
        return TestClient(app)

    async def test_complete_sub_agent_workflow(self, client):
        """Test complete sub-agent workflow"""
        auth_headers = {"Authorization": "Bearer test-token-for-testing-only"}

        # 1. Request access
        request_data = {
            "agent_id": "e2e-test-agent",
            "scopes": ["read:files", "edit:buffers", "exec:terminal"],
            "root_path": "/tmp/test_workspace",
            "reason": "End-to-end testing"
        }

        response = client.post("/mcp/request_access", json=request_data, headers=auth_headers)
        assert response.status_code == 202
        request_id = response.json()["request_id"]

        # 2. Approve access
        approve_data = {
            "request_id": request_id,
            "approved_scopes": ["read:files", "edit:buffers"]
        }

        response = client.post("/mcp/approve", json=approve_data, headers=auth_headers)
        assert response.status_code == 200
        session_data = response.json()

        # 3. WebSocket communication
        with client.websocket_connect("/ws") as websocket:
            # Authenticate
            auth_message = {
                "type": "auth",
                "token": session_data["token"]
            }
            websocket.send_json(auth_message)
            auth_response = websocket.receive_json()
            assert auth_response["type"] == "auth_success"

            # Send file read command
            command_message = {
                "type": "cmd",
                "session_id": session_data["session_id"],
                "action": "file_operation",
                "params": {
                    "operation": "read",
                    "path": "/tmp/test_workspace/test.txt"
                }
            }
            websocket.send_json(command_message)

            # Receive result
            result_response = websocket.receive_json()
            assert result_response["type"] == "result"
            assert result_response["status"] == "success"
            assert "content" in result_response["data"]
```

---

## 📊 Test Coverage

### Coverage Configuration (`pytest.ini`)

```ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --cov=bridge
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=90
    -v
```

### Coverage Report

```bash
# Run tests with coverage
pytest --cov=bridge --cov-report=html

# View coverage report
open htmlcov/index.html
```

---

## 🚀 Continuous Integration

### GitHub Actions Workflow (`.github/workflows/test.yml`)

```yaml
name: MCP Bridge Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-asyncio

      - name: Run tests
        run: |
          pytest --cov=bridge --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

---

## 📋 Testing Checklist

### Pre-release Testing

- [ ] Unit tests passing (100% coverage)
- [ ] Integration tests passing
- [ ] Security tests passing
- [ ] Performance tests within limits
- [ ] End-to-end workflow tests passing
- [ ] Load testing completed
- [ ] Security audit completed
- [ ] Documentation updated

### Regression Testing

- [ ] All existing functionality works
- [ ] No performance degradation
- [ ] Security measures intact
- [ ] API compatibility maintained
- [ ] WebSocket functionality preserved

---

**Testing strategy ini memastikan sistem MCP Bridge berfungsi dengan aman, efisien, dan dapat diandalkan untuk operasi sub-agent di VoidEditor.**
