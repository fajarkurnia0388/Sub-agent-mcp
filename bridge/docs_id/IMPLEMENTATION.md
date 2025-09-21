# MCP VoidEditor Bridge — Panduan Implementasi

> **Bahasa**: [🇺🇸 English](../IMPLEMENTATION.md) | [🇮🇩 Bahasa Indonesia](IMPLEMENTATION.md)

**Contoh kode lengkap dan detail implementasi untuk sistem MCP Bridge**

## 🏗️ Implementasi Arsitektur

MCP Bridge diimplementasikan sebagai aplikasi FastAPI dengan dukungan WebSocket, memberikan komunikasi aman antara agen Cursor dan plugin VoidEditor.

### Komponen Inti

1. **FastAPI Server** - Endpoint HTTP/WebSocket
2. **Session Manager** - Penanganan sesi efemeral
3. **WebSocket Manager** - Komunikasi real-time
4. **Security Layer** - Autentikasi & otorisasi
5. **Audit Logger** - Logging keamanan dan aktivitas

---

## 🚀 Aplikasi Utama (`bridge/main.py`)

```python
#!/usr/bin/env python3
"""
MCP VoidEditor Bridge - Aplikasi Utama
Bridge aman yang memungkinkan sub-agent beroperasi di VoidEditor
"""

import asyncio
import uvicorn
from fastapi import FastAPI, Depends, HTTPException, WebSocket, WebSocketDisconnect
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
import logging
import os
from typing import List, Optional
import yaml
from datetime import datetime

from .session_manager import SessionManager
from .websocket_manager import WebSocketManager
from .models import (
    AccessRequest, AccessResponse, SessionInfo,
    WebSocketMessage, CommandMessage, ResultMessage, EventMessage
)
from .security import SecurityManager
from .audit import AuditLogger
from .rate_limiter import RateLimiter

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize FastAPI app
app = FastAPI(
    title="MCP VoidEditor Bridge",
    description="Bridge aman untuk komunikasi Cursor-VoidEditor",
    version="1.0.0"
)

# Security
security = HTTPBearer()
security_manager = SecurityManager()
audit_logger = AuditLogger()
rate_limiter = RateLimiter()

# Managers
session_manager = SessionManager()
websocket_manager = WebSocketManager()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://127.0.0.1:3000"],  # Cursor UI
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.on_event("startup")
async def startup_event():
    """Initialize services on startup"""
    logger.info("Starting MCP VoidEditor Bridge...")
    await session_manager.initialize()
    await websocket_manager.initialize()
    audit_logger.log_system_event("bridge_started", {"timestamp": datetime.now()})
    logger.info("Bridge started successfully")

@app.on_event("shutdown")
async def shutdown_event():
    """Cleanup on shutdown"""
    logger.info("Shutting down MCP VoidEditor Bridge...")
    await websocket_manager.cleanup()
    await session_manager.cleanup()
    audit_logger.log_system_event("bridge_shutdown", {"timestamp": datetime.now()})
    logger.info("Bridge shutdown complete")

# Dependency untuk autentikasi
async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """Verify MCP token"""
    if not security_manager.verify_token(credentials.credentials):
        raise HTTPException(status_code=401, detail="Invalid token")
    return credentials.credentials

# Health check endpoint
@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "timestamp": datetime.now(),
        "active_sessions": len(session_manager.active_sessions),
        "websocket_connections": websocket_manager.connection_count
    }

# Access request endpoint
@app.post("/mcp/request_access", response_model=AccessResponse)
async def request_access(
    request: AccessRequest,
    token: str = Depends(verify_token)
):
    """Request akses ke VoidEditor untuk sub-agent"""

    # Rate limiting
    if not await rate_limiter.check_rate_limit(token):
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

    # Validate request
    if not security_manager.validate_request(request):
        raise HTTPException(status_code=400, detail="Invalid request")

    # Create pending session
    session_id = await session_manager.create_pending_session(request, token)

    # Log audit
    audit_logger.log_access_request(session_id, request, token)

    # Notify UI via SSE
    await websocket_manager.broadcast_to_ui({
        "type": "access_request",
        "session_id": session_id,
        "agent_id": request.agent_id,
        "scopes": request.scopes,
        "timestamp": datetime.now()
    })

    return AccessResponse(
        request_id=session_id,
        status="pending",
        message="Request akses dikirim untuk persetujuan pengguna"
    )

# Approve access endpoint
@app.post("/mcp/approve")
async def approve_access(
    request_id: str,
    approved_scopes: List[str],
    token: str = Depends(verify_token)
):
    """Setujui request akses"""

    # Get pending session
    session = await session_manager.get_pending_session(request_id)
    if not session:
        raise HTTPException(status_code=404, detail="Session tidak ditemukan")

    # Create active session
    active_session = await session_manager.create_active_session(
        session, approved_scopes
    )

    # Log audit
    audit_logger.log_session_approval(request_id, approved_scopes, token)

    # Notify agent
    await websocket_manager.notify_agent(active_session.agent_id, {
        "type": "session_created",
        "session_id": active_session.session_id,
        "token": active_session.token,
        "expires_at": active_session.expires_at,
        "scopes": approved_scopes
    })

    return {
        "session_id": active_session.session_id,
        "token": active_session.token,
        "expires_at": active_session.expires_at,
        "scopes": approved_scopes
    }

# WebSocket endpoint untuk komunikasi real-time
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """WebSocket endpoint untuk komunikasi agent-plugin"""

    await websocket.accept()
    connection_id = None

    try:
        # Authenticate connection
        auth_message = await websocket.receive_json()
        if auth_message.get("type") != "auth":
            await websocket.close(code=1008, reason="Authentication required")
            return

        # Verify session token
        session = await session_manager.get_session_by_token(auth_message["token"])
        if not session:
            await websocket.close(code=1008, reason="Invalid session")
            return

        # Register connection
        connection_id = await websocket_manager.register_connection(
            websocket, session.session_id
        )

        # Log connection
        audit_logger.log_websocket_connection(session.session_id, connection_id)

        # Handle messages
        while True:
            try:
                message = await websocket.receive_json()
                await handle_websocket_message(websocket, session, message)
            except WebSocketDisconnect:
                break
            except Exception as e:
                logger.error(f"WebSocket error: {e}")
                await websocket.send_json({
                    "type": "error",
                    "message": str(e)
                })

    finally:
        if connection_id:
            await websocket_manager.unregister_connection(connection_id)
            audit_logger.log_websocket_disconnection(connection_id)

async def handle_websocket_message(websocket: WebSocket, session, message):
    """Handle incoming WebSocket message"""

    message_type = message.get("type")

    if message_type == "cmd":
        # Forward command ke plugin
        await websocket_manager.forward_to_plugin(session.session_id, message)

    elif message_type == "result":
        # Forward result ke agent
        await websocket_manager.forward_to_agent(session.session_id, message)

    elif message_type == "event":
        # Forward event ke agent
        await websocket_manager.forward_to_agent(session.session_id, message)

    else:
        await websocket.send_json({
            "type": "error",
            "message": f"Unknown message type: {message_type}"
        })

# Server-Sent Events untuk notifikasi UI
@app.get("/events")
async def events_endpoint():
    """SSE endpoint untuk notifikasi real-time ke UI"""

    async def event_generator():
        while True:
            # Check for new events
            event = await websocket_manager.get_ui_event()
            if event:
                yield f"data: {event}\n\n"
            await asyncio.sleep(0.1)

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )

if __name__ == "__main__":
    uvicorn.run(
        "bridge.main:app",
        host="127.0.0.1",
        port=8787,
        reload=False,
        log_level="info"
    )
```

---

## 🔐 Security Manager (`bridge/security.py`)

```python
"""
Security Manager untuk MCP Bridge
Menangani autentikasi, otorisasi, dan validasi keamanan
"""

import secrets
import hashlib
import hmac
from typing import List, Dict, Any
from datetime import datetime, timedelta
import yaml
import os

class SecurityManager:
    def __init__(self):
        self.mcp_token = os.getenv("MCP_BRIDGE_TOKEN")
        self.allowed_roots = os.getenv("ALLOWED_ROOTS", "/workspace").split(",")
        self.load_security_config()

    def load_security_config(self):
        """Load konfigurasi keamanan dari file"""
        try:
            with open("security.yaml", "r") as f:
                self.config = yaml.safe_load(f)
        except FileNotFoundError:
            self.config = self.get_default_config()

    def get_default_config(self):
        """Default security configuration"""
        return {
            "security": {
                "token_length": 32,
                "token_charset": "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789",
                "max_sessions_per_agent": 3,
                "session_cleanup_interval": 60,
                "rate_limit": {
                    "requests_per_minute": 100,
                    "burst_allowance": 20
                },
                "filesystem": {
                    "max_file_size": 10485760,  # 10MB
                    "allowed_extensions": [".js", ".ts", ".py", ".md", ".json", ".yaml", ".yml"],
                    "blocked_paths": ["/etc", "/usr", "/bin", "/sbin"]
                }
            }
        }

    def verify_token(self, token: str) -> bool:
        """Verify MCP token"""
        return hmac.compare_digest(token, self.mcp_token)

    def generate_session_token(self) -> str:
        """Generate session token yang aman"""
        charset = self.config["security"]["token_charset"]
        length = self.config["security"]["token_length"]
        return ''.join(secrets.choice(charset) for _ in range(length))

    def validate_request(self, request) -> bool:
        """Validate access request"""
        # Validate scopes
        if not self.validate_scopes(request.scopes):
            return False

        # Validate root path
        if not self.validate_root_path(request.root_path):
            return False

        # Validate TTL
        if request.ttl and request.ttl > 3600:  # Max 1 hour
            return False

        return True

    def validate_scopes(self, scopes: List[str]) -> bool:
        """Validate requested scopes"""
        valid_scopes = {
            "read:files", "write:files", "delete:files",
            "read:buffers", "edit:buffers", "create:buffers",
            "exec:terminal", "read:terminal",
            "git:read", "git:write",
            "read:project", "build:project", "test:project"
        }

        for scope in scopes:
            if scope not in valid_scopes:
                return False

        return True

    def validate_root_path(self, path: str) -> bool:
        """Validate root path"""
        import os
        abs_path = os.path.abspath(path)

        for allowed_root in self.allowed_roots:
            if abs_path.startswith(os.path.abspath(allowed_root)):
                return True

        return False

    def validate_file_operation(self, operation: str, path: str, content: str = None) -> bool:
        """Validate file operation"""
        # Check path
        if not self.validate_root_path(path):
            return False

        # Check file extension
        if not self.validate_file_extension(path):
            return False

        # Check file size
        if content and len(content) > self.config["security"]["filesystem"]["max_file_size"]:
            return False

        # Check blocked paths
        if self.is_blocked_path(path):
            return False

        return True

    def validate_file_extension(self, path: str) -> bool:
        """Validate file extension"""
        allowed_extensions = self.config["security"]["filesystem"]["allowed_extensions"]
        _, ext = os.path.splitext(path)
        return ext in allowed_extensions

    def is_blocked_path(self, path: str) -> bool:
        """Check if path is blocked"""
        blocked_paths = self.config["security"]["filesystem"]["blocked_paths"]
        import os
        abs_path = os.path.abspath(path)

        for blocked in blocked_paths:
            if abs_path.startswith(blocked):
                return True

        return False
```

---

## 📊 Session Manager (`bridge/session_manager.py`)

```python
"""
Session Manager untuk MCP Bridge
Menangani lifecycle sesi efemeral
"""

import asyncio
from typing import Dict, Optional, List
from datetime import datetime, timedelta
import uuid
from dataclasses import dataclass, asdict

@dataclass
class Session:
    session_id: str
    agent_id: str
    token: str
    scopes: List[str]
    root_path: str
    created_at: datetime
    expires_at: datetime
    status: str  # "pending", "active", "expired"
    last_activity: datetime

class SessionManager:
    def __init__(self):
        self.active_sessions: Dict[str, Session] = {}
        self.pending_sessions: Dict[str, Session] = {}
        self.cleanup_task = None

    async def initialize(self):
        """Initialize session manager"""
        self.cleanup_task = asyncio.create_task(self.cleanup_expired_sessions())

    async def cleanup(self):
        """Cleanup session manager"""
        if self.cleanup_task:
            self.cleanup_task.cancel()
            try:
                await self.cleanup_task
            except asyncio.CancelledError:
                pass

    async def create_pending_session(self, request, token: str) -> str:
        """Create pending session"""
        session_id = str(uuid.uuid4())

        session = Session(
            session_id=session_id,
            agent_id=request.agent_id,
            token="",  # Will be set when approved
            scopes=request.scopes,
            root_path=request.root_path,
            created_at=datetime.now(),
            expires_at=datetime.now() + timedelta(seconds=request.ttl or 300),
            status="pending",
            last_activity=datetime.now()
        )

        self.pending_sessions[session_id] = session
        return session_id

    async def create_active_session(self, pending_session: Session, approved_scopes: List[str]) -> Session:
        """Create active session from pending session"""
        from .security import SecurityManager

        security_manager = SecurityManager()
        session_token = security_manager.generate_session_token()

        active_session = Session(
            session_id=pending_session.session_id,
            agent_id=pending_session.agent_id,
            token=session_token,
            scopes=approved_scopes,
            root_path=pending_session.root_path,
            created_at=datetime.now(),
            expires_at=pending_session.expires_at,
            status="active",
            last_activity=datetime.now()
        )

        # Remove from pending and add to active
        del self.pending_sessions[pending_session.session_id]
        self.active_sessions[session_id] = active_session

        return active_session

    async def get_session_by_token(self, token: str) -> Optional[Session]:
        """Get session by token"""
        for session in self.active_sessions.values():
            if session.token == token:
                return session
        return None

    async def get_pending_session(self, session_id: str) -> Optional[Session]:
        """Get pending session"""
        return self.pending_sessions.get(session_id)

    async def update_activity(self, session_id: str):
        """Update session last activity"""
        if session_id in self.active_sessions:
            self.active_sessions[session_id].last_activity = datetime.now()

    async def revoke_session(self, session_id: str):
        """Revoke session"""
        if session_id in self.active_sessions:
            del self.active_sessions[session_id]

    async def cleanup_expired_sessions(self):
        """Cleanup expired sessions"""
        while True:
            try:
                current_time = datetime.now()
                expired_sessions = []

                # Check active sessions
                for session_id, session in self.active_sessions.items():
                    if current_time > session.expires_at:
                        expired_sessions.append(session_id)

                # Check pending sessions
                for session_id, session in self.pending_sessions.items():
                    if current_time > session.expires_at:
                        expired_sessions.append(session_id)

                # Remove expired sessions
                for session_id in expired_sessions:
                    if session_id in self.active_sessions:
                        del self.active_sessions[session_id]
                    if session_id in self.pending_sessions:
                        del self.pending_sessions[session_id]

                # Sleep for cleanup interval
                await asyncio.sleep(60)  # Cleanup every minute

            except asyncio.CancelledError:
                break
            except Exception as e:
                print(f"Error in cleanup: {e}")
                await asyncio.sleep(60)
```

---

## 🌐 WebSocket Manager (`bridge/websocket_manager.py`)

```python
"""
WebSocket Manager untuk MCP Bridge
Menangani komunikasi real-time antara agent dan plugin
"""

import asyncio
import json
from typing import Dict, List, Optional
from fastapi import WebSocket
from datetime import datetime

class WebSocketManager:
    def __init__(self):
        self.connections: Dict[str, WebSocket] = {}
        self.agent_connections: Dict[str, str] = {}  # agent_id -> connection_id
        self.plugin_connections: Dict[str, str] = {}  # session_id -> connection_id
        self.ui_events: List[dict] = []

    async def initialize(self):
        """Initialize WebSocket manager"""
        pass

    async def cleanup(self):
        """Cleanup WebSocket manager"""
        # Close all connections
        for connection_id, websocket in self.connections.items():
            try:
                await websocket.close()
            except:
                pass
        self.connections.clear()
        self.agent_connections.clear()
        self.plugin_connections.clear()

    async def register_connection(self, websocket: WebSocket, session_id: str) -> str:
        """Register WebSocket connection"""
        connection_id = f"conn_{len(self.connections)}"
        self.connections[connection_id] = websocket

        # Determine if this is agent or plugin connection
        # For now, assume it's agent connection
        self.agent_connections[session_id] = connection_id

        return connection_id

    async def unregister_connection(self, connection_id: str):
        """Unregister WebSocket connection"""
        if connection_id in self.connections:
            del self.connections[connection_id]

        # Remove from agent connections
        for session_id, conn_id in list(self.agent_connections.items()):
            if conn_id == connection_id:
                del self.agent_connections[session_id]
                break

        # Remove from plugin connections
        for session_id, conn_id in list(self.plugin_connections.items()):
            if conn_id == connection_id:
                del self.plugin_connections[session_id]
                break

    async def send_to_connection(self, connection_id: str, message: dict):
        """Send message to specific connection"""
        if connection_id in self.connections:
            websocket = self.connections[connection_id]
            try:
                await websocket.send_json(message)
            except Exception as e:
                print(f"Error sending message: {e}")
                await self.unregister_connection(connection_id)

    async def forward_to_plugin(self, session_id: str, message: dict):
        """Forward message to plugin"""
        if session_id in self.plugin_connections:
            connection_id = self.plugin_connections[session_id]
            await self.send_to_connection(connection_id, message)

    async def forward_to_agent(self, session_id: str, message: dict):
        """Forward message to agent"""
        if session_id in self.agent_connections:
            connection_id = self.agent_connections[session_id]
            await self.send_to_connection(connection_id, message)

    async def notify_agent(self, agent_id: str, message: dict):
        """Notify specific agent"""
        # Find connection by agent_id
        for session_id, connection_id in self.agent_connections.items():
            # This is simplified - in real implementation, you'd need to track agent_id
            await self.send_to_connection(connection_id, message)
            break

    async def broadcast_to_ui(self, event: dict):
        """Broadcast event to UI via SSE"""
        self.ui_events.append(event)

    async def get_ui_event(self) -> Optional[dict]:
        """Get UI event for SSE"""
        if self.ui_events:
            return self.ui_events.pop(0)
        return None

    @property
    def connection_count(self) -> int:
        """Get connection count"""
        return len(self.connections)
```

---

## 📝 Models (`bridge/models.py`)

```python
"""
Data models untuk MCP Bridge
"""

from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
from datetime import datetime
from enum import Enum

class SessionStatus(str, Enum):
    PENDING = "pending"
    ACTIVE = "active"
    EXPIRED = "expired"
    REVOKED = "revoked"

class AccessRequest(BaseModel):
    agent_id: str = Field(..., description="Identifier unik untuk agent")
    scopes: List[str] = Field(..., description="Daftar izin yang diminta")
    root_path: str = Field(..., description="Path root yang diizinkan")
    ttl: Optional[int] = Field(300, description="Time-to-live dalam detik")
    reason: Optional[str] = Field(None, description="Alasan request akses")

class AccessResponse(BaseModel):
    request_id: str = Field(..., description="ID request")
    status: str = Field(..., description="Status request")
    message: str = Field(..., description="Pesan response")

class SessionInfo(BaseModel):
    session_id: str
    agent_id: str
    scopes: List[str]
    root_path: str
    created_at: datetime
    expires_at: datetime
    status: SessionStatus
    last_activity: datetime

class WebSocketMessage(BaseModel):
    type: str = Field(..., description="Tipe pesan")
    session_id: str = Field(..., description="ID sesi")
    timestamp: datetime = Field(default_factory=datetime.now)

class CommandMessage(WebSocketMessage):
    type: str = "cmd"
    action: str = Field(..., description="Aksi yang diminta")
    params: Dict[str, Any] = Field(..., description="Parameter aksi")

class ResultMessage(WebSocketMessage):
    type: str = "result"
    request_id: str = Field(..., description="ID request")
    status: str = Field(..., description="Status hasil")
    data: Optional[Dict[str, Any]] = Field(None, description="Data hasil")
    error: Optional[str] = Field(None, description="Pesan error")

class EventMessage(WebSocketMessage):
    type: str = "event"
    event_type: str = Field(..., description="Tipe event")
    data: Dict[str, Any] = Field(..., description="Data event")
```

---

**Implementasi ini memberikan foundation yang solid untuk sistem MCP Bridge yang aman dan efisien, memungkinkan sub-agent beroperasi dengan kemampuan IDE penuh di VoidEditor.**
