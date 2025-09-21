# MCP VoidEditor Bridge — Implementation Guide

> **Language**: [🇺🇸 English](IMPLEMENTATION.md) | [🇮🇩 Bahasa Indonesia](docs_id/IMPLEMENTATION.md)

**Complete code examples and implementation details for the MCP Bridge system**

## 🏗️ Architecture Implementation

The MCP Bridge is implemented as a FastAPI application with WebSocket support, providing secure communication between Cursor agents and VoidEditor plugins.

### Core Components

1. **FastAPI Server** - HTTP/WebSocket endpoints
2. **Session Manager** - Ephemeral session handling
3. **WebSocket Manager** - Real-time communication
4. **Security Layer** - Authentication & authorization
5. **Audit Logger** - Security and activity logging

---

## 🚀 Main Application (`bridge/main.py`)

```python
#!/usr/bin/env python3
"""
MCP VoidEditor Bridge - Main Application
Secure bridge enabling sub-agents to operate in VoidEditor
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

# Load configuration
def load_config():
    config_path = os.getenv("MCP_CONFIG", "config/config.yaml")
    with open(config_path, 'r') as f:
        return yaml.safe_load(f)

config = load_config()

# Initialize logging
logging.basicConfig(
    level=config['logging']['level'],
    format=config['logging']['format']
)
logger = logging.getLogger(__name__)

# Initialize core components
app = FastAPI(
    title="MCP VoidEditor Bridge",
    description="Secure bridge for sub-agents in VoidEditor",
    version="1.0.0",
    docs_url="/docs" if config['bridge']['debug'] else None
)

# Security setup
security = HTTPBearer()
security_manager = SecurityManager(config['security'])
audit_logger = AuditLogger()
rate_limiter = RateLimiter(
    requests_per_window=config['rate_limiting']['requests_per_window'],
    window_seconds=config['rate_limiting']['window_seconds']
)

# Core managers
session_manager = SessionManager(config['sessions'])
websocket_manager = WebSocketManager()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=config['security']['allowed_origins'],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

# Authentication dependency
async def verify_mcp_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    if not security_manager.verify_mcp_token(credentials.credentials):
        audit_logger.log_security_event("invalid_mcp_token", credentials.credentials[:10])
        raise HTTPException(status_code=401, detail="Invalid MCP token")
    return credentials.credentials

# Rate limiting dependency
async def check_rate_limit(request: any):
    client_id = getattr(request, 'client', {}).get('host', 'unknown')
    if not await rate_limiter.is_allowed(client_id):
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

# ============================================================================
# Management API Endpoints
# ============================================================================

@app.post("/mcp/request_access", response_model=AccessResponse)
async def request_access(
    request: AccessRequest,
    _: str = Depends(verify_mcp_token),
    __: any = Depends(check_rate_limit)
):
    """Request access to VoidEditor for sub-agent operations"""
    try:
        # Validate scopes
        invalid_scopes = security_manager.validate_scopes(request.scopes)
        if invalid_scopes:
            audit_logger.log_access_request(
                request.agent_id, request.scopes, "denied_invalid_scopes"
            )
            raise HTTPException(
                status_code=400,
                detail=f"Invalid scopes: {invalid_scopes}"
            )

        # Create access request
        request_id = await session_manager.create_access_request(
            agent_id=request.agent_id,
            scopes=request.scopes,
            roots=request.roots,
            reason=request.reason
        )

        audit_logger.log_access_request(request.agent_id, request.scopes, "created")

        return AccessResponse(
            request_id=request_id,
            status="pending",
            created_at=datetime.utcnow()
        )

    except Exception as e:
        logger.error(f"Failed to create access request: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.get("/mcp/requests")
async def list_requests(
    status: Optional[str] = None,
    limit: int = 50,
    offset: int = 0,
    _: str = Depends(verify_mcp_token)
):
    """List recent access requests"""
    try:
        requests = await session_manager.list_requests(
            status=status, limit=limit, offset=offset
        )
        return {
            "requests": requests,
            "total": len(requests),
            "has_more": len(requests) == limit
        }
    except Exception as e:
        logger.error(f"Failed to list requests: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.post("/mcp/approve")
async def approve_request(
    request_data: dict,
    _: str = Depends(verify_mcp_token)
):
    """Approve an access request and create session"""
    try:
        request_id = request_data["request_id"]
        approved_scopes = request_data.get("approved_scopes")
        ttl_seconds = request_data.get("ttl_seconds", 300)

        # Get original request
        access_request = await session_manager.get_request(request_id)
        if not access_request:
            raise HTTPException(status_code=404, detail="Request not found")

        # Create session
        session_info = await session_manager.approve_request(
            request_id=request_id,
            approved_scopes=approved_scopes or access_request.scopes,
            ttl_seconds=ttl_seconds
        )

        audit_logger.log_session_created(
            session_info.session_id,
            access_request.agent_id,
            session_info.approved_scopes
        )

        return {
            "session_id": session_info.session_id,
            "session_token": session_info.session_token,
            "expires_at": session_info.expires_at,
            "approved_scopes": session_info.approved_scopes
        }

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Failed to approve request: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.post("/mcp/deny")
async def deny_request(
    request_data: dict,
    _: str = Depends(verify_mcp_token)
):
    """Deny an access request"""
    try:
        request_id = request_data["request_id"]
        reason = request_data.get("reason", "No reason provided")

        await session_manager.deny_request(request_id, reason)
        audit_logger.log_request_denied(request_id, reason)

        return {
            "request_id": request_id,
            "status": "denied",
            "denied_at": datetime.utcnow()
        }

    except Exception as e:
        logger.error(f"Failed to deny request: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.post("/mcp/revoke")
async def revoke_session(
    session_data: dict,
    _: str = Depends(verify_mcp_token)
):
    """Revoke an active session"""
    try:
        session_id = session_data["session_id"]
        reason = session_data.get("reason", "Manual revocation")

        await session_manager.revoke_session(session_id, reason)
        audit_logger.log_session_revoked(session_id, reason)

        return {
            "session_id": session_id,
            "status": "revoked",
            "revoked_at": datetime.utcnow()
        }

    except Exception as e:
        logger.error(f"Failed to revoke session: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.get("/mcp/sessions")
async def list_sessions(_: str = Depends(verify_mcp_token)):
    """List active sessions"""
    try:
        sessions = await session_manager.list_active_sessions()
        return {"sessions": sessions, "total": len(sessions)}

    except Exception as e:
        logger.error(f"Failed to list sessions: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

# ============================================================================
# WebSocket Endpoints
# ============================================================================

@app.websocket("/mcp/session/{session_id}")
async def agent_websocket(websocket: WebSocket, session_id: str):
    """WebSocket endpoint for agent communication"""
    try:
        # Accept connection
        await websocket.accept()

        # Authenticate session
        auth_message = await websocket.receive_json()
        session_token = auth_message.get("session_token")

        if not await session_manager.verify_session(session_id, session_token):
            await websocket.close(code=4008, reason="Invalid session")
            return

        # Register agent connection
        await websocket_manager.register_agent(session_id, websocket)
        logger.info(f"Agent connected for session: {session_id}")

        # Handle messages
        while True:
            try:
                data = await websocket.receive_json()
                message = WebSocketMessage(**data)

                # Process message based on type
                if message.type == "cmd":
                    await handle_agent_command(session_id, message)
                else:
                    logger.warning(f"Unknown message type: {message.type}")

            except WebSocketDisconnect:
                logger.info(f"Agent disconnected from session: {session_id}")
                break
            except Exception as e:
                logger.error(f"Error handling agent message: {e}")
                await websocket.send_json({
                    "type": "error",
                    "error": str(e)
                })

    except Exception as e:
        logger.error(f"Agent WebSocket error: {e}")
    finally:
        await websocket_manager.unregister_agent(session_id)

@app.websocket("/plugin/{plugin_id}")
async def plugin_websocket(websocket: WebSocket, plugin_id: str):
    """WebSocket endpoint for plugin registration and communication"""
    try:
        await websocket.accept()

        # Handle plugin registration
        registration = await websocket.receive_json()

        if registration.get("type") != "register":
            await websocket.close(code=4000, reason="Expected registration")
            return

        # Register plugin
        await websocket_manager.register_plugin(plugin_id, websocket)
        logger.info(f"Plugin registered: {plugin_id}")

        # Send registration confirmation
        await websocket.send_json({
            "type": "registered",
            "status": "ok",
            "assigned_id": plugin_id,
            "bridge_version": "1.0.0"
        })

        # Handle plugin messages
        while True:
            try:
                data = await websocket.receive_json()
                await handle_plugin_message(plugin_id, data)

            except WebSocketDisconnect:
                logger.info(f"Plugin disconnected: {plugin_id}")
                break
            except Exception as e:
                logger.error(f"Error handling plugin message: {e}")

    except Exception as e:
        logger.error(f"Plugin WebSocket error: {e}")
    finally:
        await websocket_manager.unregister_plugin(plugin_id)

# ============================================================================
# Health & Monitoring Endpoints
# ============================================================================

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0",
        "active_sessions": len(session_manager.sessions),
        "active_plugins": len(websocket_manager.plugins)
    }

@app.get("/metrics")
async def get_metrics(_: str = Depends(verify_mcp_token)):
    """Metrics endpoint for monitoring"""
    return {
        "sessions": {
            "active": len(session_manager.sessions),
            "total_created": session_manager.total_created,
            "total_expired": session_manager.total_expired
        },
        "plugins": {
            "active": len(websocket_manager.plugins),
            "total_registered": websocket_manager.total_registered
        },
        "requests": {
            "total": rate_limiter.total_requests,
            "rejected": rate_limiter.rejected_requests
        }
    }

# ============================================================================
# Message Handlers
# ============================================================================

async def handle_agent_command(session_id: str, message: WebSocketMessage):
    """Handle command from agent"""
    try:
        # Validate session and permissions
        session = await session_manager.get_session(session_id)
        if not session:
            raise ValueError("Session not found")

        # Create command message
        command = CommandMessage(
            type="cmd",
            id=message.id,
            action=message.action,
            args=message.args
        )

        # Validate scope permissions
        required_scope = get_required_scope(command.action)
        if required_scope not in session.approved_scopes:
            raise PermissionError(f"Action requires scope: {required_scope}")

        # Forward to plugin
        await websocket_manager.send_to_plugin(command.dict())

        # Log command
        audit_logger.log_command(session_id, command.action, "forwarded")

    except Exception as e:
        logger.error(f"Failed to handle agent command: {e}")
        await websocket_manager.send_to_agent(session_id, {
            "type": "result",
            "id": message.id,
            "status": "error",
            "error": str(e)
        })

async def handle_plugin_message(plugin_id: str, message: dict):
    """Handle message from plugin"""
    try:
        message_type = message.get("type")

        if message_type == "result":
            # Forward result to agent
            session_id = message.get("session_id")
            if session_id:
                await websocket_manager.send_to_agent(session_id, message)

        elif message_type == "event":
            # Forward event to agent
            session_id = message.get("session_id")
            if session_id:
                await websocket_manager.send_to_agent(session_id, message)

        else:
            logger.warning(f"Unknown plugin message type: {message_type}")

    except Exception as e:
        logger.error(f"Failed to handle plugin message: {e}")

def get_required_scope(action: str) -> str:
    """Map action to required scope"""
    scope_mapping = {
        # File operations
        "open_file": "read:files",
        "create_file": "write:files",
        "delete_file": "delete:files",

        # Buffer operations
        "edit_buffer": "edit:buffers",
        "get_buffers": "read:buffers",

        # Code analysis
        "analyze_syntax": "analyze:syntax",
        "get_completions": "analyze:syntax",

        # Project operations
        "explore_tree": "explore:project",
        "build_project": "build:project",

        # Terminal operations
        "exec_command": "exec:terminal",

        # Git operations
        "git_status": "git:status",
        "git_commit": "git:commit",
    }

    return scope_mapping.get(action, "unknown:scope")

# ============================================================================
# Startup & Shutdown
# ============================================================================

@app.on_event("startup")
async def startup_event():
    """Initialize components on startup"""
    logger.info("Starting MCP VoidEditor Bridge")

    # Start session cleanup task
    asyncio.create_task(session_manager.cleanup_task())

    # Start rate limiter cleanup
    asyncio.create_task(rate_limiter.cleanup_task())

    logger.info("Bridge started successfully")

@app.on_event("shutdown")
async def shutdown_event():
    """Cleanup on shutdown"""
    logger.info("Shutting down MCP VoidEditor Bridge")

    # Close all sessions
    await session_manager.close_all_sessions()

    # Close all WebSocket connections
    await websocket_manager.close_all_connections()

    logger.info("Bridge shut down successfully")

# ============================================================================
# Main Entry Point
# ============================================================================

def main():
    """Main entry point"""
    host = config['bridge']['host']
    port = config['bridge']['port']
    debug = config['bridge']['debug']

    logger.info(f"Starting bridge on {host}:{port}")

    uvicorn.run(
        "bridge.main:app",
        host=host,
        port=port,
        reload=debug,
        log_level="info" if not debug else "debug"
    )

if __name__ == "__main__":
    main()
```

---

## 🔐 Security Manager (`bridge/security.py`)

```python
"""
Security Manager - Authentication and authorization
"""

import jwt
import secrets
import hashlib
from datetime import datetime, timedelta
from typing import List, Optional, Set
import os

class SecurityManager:
    def __init__(self, config: dict):
        self.config = config
        self.mcp_token_hash = self._hash_token(config['mcp_token'])
        self.session_secret = secrets.token_urlsafe(32)

        # Valid scopes
        self.valid_scopes = {
            # File operations
            "read:files", "write:files", "create:files", "delete:files",
            "rename:files", "search:files",

            # Buffer management
            "read:buffers", "edit:buffers", "manage:buffers",
            "split:panes", "manage:tabs",

            # Code analysis
            "analyze:syntax", "detect:errors", "suggest:completion",
            "format:code", "navigate:symbols",

            # Project management
            "explore:project", "manage:workspace", "watch:changes",
            "build:project",

            # Terminal & execution
            "exec:terminal", "run:commands", "debug:session", "test:code",

            # Version control
            "git:status", "git:commit", "git:branch", "view:diffs",
        }

    def _hash_token(self, token: str) -> str:
        """Hash token for secure comparison"""
        return hashlib.sha256(token.encode()).hexdigest()

    def verify_mcp_token(self, token: str) -> bool:
        """Verify MCP token"""
        return secrets.compare_digest(
            self._hash_token(token),
            self.mcp_token_hash
        )

    def validate_scopes(self, scopes: List[str]) -> List[str]:
        """Validate requested scopes, return invalid ones"""
        return [scope for scope in scopes if scope not in self.valid_scopes]

    def generate_session_token(self, session_id: str, agent_id: str,
                             scopes: List[str], ttl_seconds: int) -> str:
        """Generate JWT session token"""
        payload = {
            "session_id": session_id,
            "agent_id": agent_id,
            "scopes": scopes,
            "iat": datetime.utcnow(),
            "exp": datetime.utcnow() + timedelta(seconds=ttl_seconds)
        }

        return jwt.encode(payload, self.session_secret, algorithm="HS256")

    def verify_session_token(self, token: str) -> Optional[dict]:
        """Verify and decode session token"""
        try:
            payload = jwt.decode(
                token, self.session_secret, algorithms=["HS256"]
            )
            return payload
        except jwt.InvalidTokenError:
            return None

    def check_scope_permission(self, user_scopes: List[str],
                             required_scope: str) -> bool:
        """Check if user has required scope"""
        return required_scope in user_scopes
```

---

## 📋 Session Manager (`bridge/session_manager.py`)

```python
"""
Session Manager - Handle ephemeral sessions and access requests
"""

import asyncio
import uuid
from datetime import datetime, timedelta
from typing import Dict, List, Optional
import logging

from .models import AccessRequest, SessionInfo, RequestStatus
from .security import SecurityManager

logger = logging.getLogger(__name__)

class SessionManager:
    def __init__(self, config: dict):
        self.config = config
        self.sessions: Dict[str, SessionInfo] = {}
        self.requests: Dict[str, AccessRequest] = {}
        self.total_created = 0
        self.total_expired = 0

        # Cleanup interval
        self.cleanup_interval = config.get('cleanup_interval', 60)
        self.max_idle_time = config.get('max_idle_time', 600)
        self.max_sessions = config.get('max_sessions', 10)

    async def create_access_request(self, agent_id: str, scopes: List[str],
                                  roots: List[str], reason: str) -> str:
        """Create new access request"""
        request_id = f"req-{uuid.uuid4()}"

        request = AccessRequest(
            request_id=request_id,
            agent_id=agent_id,
            scopes=scopes,
            roots=roots,
            reason=reason,
            status=RequestStatus.PENDING,
            created_at=datetime.utcnow()
        )

        self.requests[request_id] = request
        logger.info(f"Created access request: {request_id} for agent: {agent_id}")

        return request_id

    async def get_request(self, request_id: str) -> Optional[AccessRequest]:
        """Get access request by ID"""
        return self.requests.get(request_id)

    async def list_requests(self, status: Optional[str] = None,
                          limit: int = 50, offset: int = 0) -> List[dict]:
        """List access requests with optional filtering"""
        requests = list(self.requests.values())

        if status:
            requests = [r for r in requests if r.status == status]

        # Sort by creation time (newest first)
        requests.sort(key=lambda x: x.created_at, reverse=True)

        # Apply pagination
        return [r.dict() for r in requests[offset:offset + limit]]

    async def approve_request(self, request_id: str, approved_scopes: List[str],
                            ttl_seconds: int) -> SessionInfo:
        """Approve access request and create session"""
        request = self.requests.get(request_id)
        if not request:
            raise ValueError(f"Request not found: {request_id}")

        if request.status != RequestStatus.PENDING:
            raise ValueError(f"Request not pending: {request_id}")

        # Check session limits
        if len(self.sessions) >= self.max_sessions:
            raise ValueError("Maximum sessions reached")

        # Create session
        session_id = f"sess-{uuid.uuid4()}"
        session_token = SecurityManager.generate_session_token(
            session_id, request.agent_id, approved_scopes, ttl_seconds
        )

        session_info = SessionInfo(
            session_id=session_id,
            request_id=request_id,
            agent_id=request.agent_id,
            session_token=session_token,
            approved_scopes=approved_scopes,
            allowed_roots=request.roots,
            status="active",
            created_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + timedelta(seconds=ttl_seconds),
            last_activity=datetime.utcnow()
        )

        # Update request status
        request.status = RequestStatus.APPROVED
        request.approved_at = datetime.utcnow()
        request.session_id = session_id

        # Store session
        self.sessions[session_id] = session_info
        self.total_created += 1

        logger.info(f"Approved request {request_id}, created session {session_id}")
        return session_info

    async def deny_request(self, request_id: str, reason: str):
        """Deny access request"""
        request = self.requests.get(request_id)
        if not request:
            raise ValueError(f"Request not found: {request_id}")

        request.status = RequestStatus.DENIED
        request.denied_at = datetime.utcnow()
        request.denial_reason = reason

        logger.info(f"Denied request {request_id}: {reason}")

    async def get_session(self, session_id: str) -> Optional[SessionInfo]:
        """Get session by ID"""
        return self.sessions.get(session_id)

    async def verify_session(self, session_id: str, session_token: str) -> bool:
        """Verify session exists and token is valid"""
        session = self.sessions.get(session_id)
        if not session:
            return False

        if session.status != "active":
            return False

        if datetime.utcnow() > session.expires_at:
            await self.expire_session(session_id)
            return False

        # Verify token (simplified - in real implementation, use JWT)
        if session.session_token != session_token:
            return False

        # Update last activity
        session.last_activity = datetime.utcnow()
        return True

    async def revoke_session(self, session_id: str, reason: str):
        """Revoke active session"""
        session = self.sessions.get(session_id)
        if not session:
            raise ValueError(f"Session not found: {session_id}")

        session.status = "revoked"
        session.revoked_at = datetime.utcnow()
        session.revocation_reason = reason

        logger.info(f"Revoked session {session_id}: {reason}")

    async def expire_session(self, session_id: str):
        """Mark session as expired"""
        session = self.sessions.get(session_id)
        if session:
            session.status = "expired"
            self.total_expired += 1
            logger.info(f"Session expired: {session_id}")

    async def list_active_sessions(self) -> List[dict]:
        """List all active sessions"""
        active_sessions = [
            s for s in self.sessions.values()
            if s.status == "active" and datetime.utcnow() <= s.expires_at
        ]
        return [s.dict() for s in active_sessions]

    async def close_all_sessions(self):
        """Close all active sessions"""
        for session_id in list(self.sessions.keys()):
            await self.revoke_session(session_id, "Bridge shutdown")

    async def cleanup_task(self):
        """Background task to cleanup expired sessions"""
        while True:
            try:
                await asyncio.sleep(self.cleanup_interval)
                await self._cleanup_expired_sessions()
            except Exception as e:
                logger.error(f"Session cleanup error: {e}")

    async def _cleanup_expired_sessions(self):
        """Remove expired sessions"""
        now = datetime.utcnow()
        expired_sessions = []

        for session_id, session in self.sessions.items():
            if (session.status == "active" and now > session.expires_at) or \
               (session.status != "active" and
                (now - session.last_activity).seconds > self.max_idle_time):
                expired_sessions.append(session_id)

        for session_id in expired_sessions:
            await self.expire_session(session_id)
            del self.sessions[session_id]

        if expired_sessions:
            logger.info(f"Cleaned up {len(expired_sessions)} expired sessions")
```

---

## 🔌 WebSocket Manager (`bridge/websocket_manager.py`)

```python
"""
WebSocket Manager - Handle real-time communication
"""

import asyncio
import json
import logging
from typing import Dict, Optional, List
from fastapi import WebSocket

logger = logging.getLogger(__name__)

class WebSocketManager:
    def __init__(self):
        self.agents: Dict[str, WebSocket] = {}  # session_id -> websocket
        self.plugins: Dict[str, WebSocket] = {}  # plugin_id -> websocket
        self.total_registered = 0

    async def register_agent(self, session_id: str, websocket: WebSocket):
        """Register agent WebSocket connection"""
        self.agents[session_id] = websocket
        logger.info(f"Registered agent for session: {session_id}")

    async def unregister_agent(self, session_id: str):
        """Unregister agent WebSocket connection"""
        if session_id in self.agents:
            del self.agents[session_id]
            logger.info(f"Unregistered agent for session: {session_id}")

    async def register_plugin(self, plugin_id: str, websocket: WebSocket):
        """Register plugin WebSocket connection"""
        self.plugins[plugin_id] = websocket
        self.total_registered += 1
        logger.info(f"Registered plugin: {plugin_id}")

    async def unregister_plugin(self, plugin_id: str):
        """Unregister plugin WebSocket connection"""
        if plugin_id in self.plugins:
            del self.plugins[plugin_id]
            logger.info(f"Unregistered plugin: {plugin_id}")

    async def send_to_agent(self, session_id: str, message: dict) -> bool:
        """Send message to agent"""
        websocket = self.agents.get(session_id)
        if not websocket:
            logger.warning(f"No agent found for session: {session_id}")
            return False

        try:
            await websocket.send_json(message)
            return True
        except Exception as e:
            logger.error(f"Failed to send to agent {session_id}: {e}")
            await self.unregister_agent(session_id)
            return False

    async def send_to_plugin(self, message: dict, plugin_id: Optional[str] = None) -> bool:
        """Send message to plugin(s)"""
        if plugin_id:
            # Send to specific plugin
            websocket = self.plugins.get(plugin_id)
            if not websocket:
                logger.warning(f"No plugin found: {plugin_id}")
                return False

            try:
                await websocket.send_json(message)
                return True
            except Exception as e:
                logger.error(f"Failed to send to plugin {plugin_id}: {e}")
                await self.unregister_plugin(plugin_id)
                return False
        else:
            # Broadcast to all plugins
            success_count = 0
            for pid, websocket in list(self.plugins.items()):
                try:
                    await websocket.send_json(message)
                    success_count += 1
                except Exception as e:
                    logger.error(f"Failed to send to plugin {pid}: {e}")
                    await self.unregister_plugin(pid)

            return success_count > 0

    async def broadcast_event(self, event: dict):
        """Broadcast event to all connected agents"""
        for session_id in list(self.agents.keys()):
            await self.send_to_agent(session_id, event)

    async def close_all_connections(self):
        """Close all WebSocket connections"""
        # Close agent connections
        for session_id, websocket in list(self.agents.items()):
            try:
                await websocket.close()
            except Exception as e:
                logger.error(f"Error closing agent connection {session_id}: {e}")

        # Close plugin connections
        for plugin_id, websocket in list(self.plugins.items()):
            try:
                await websocket.close()
            except Exception as e:
                logger.error(f"Error closing plugin connection {plugin_id}: {e}")

        self.agents.clear()
        self.plugins.clear()
        logger.info("Closed all WebSocket connections")
```

---

## 📊 Data Models (`bridge/models.py`)

```python
"""
Data Models - Pydantic models for API requests/responses
"""

from pydantic import BaseModel, Field, validator
from typing import List, Optional, Dict, Any
from datetime import datetime
from enum import Enum
import re

class RequestStatus(str, Enum):
    PENDING = "pending"
    APPROVED = "approved"
    DENIED = "denied"

class AccessRequest(BaseModel):
    """Access request from agent"""
    agent_id: str = Field(..., description="Unique agent identifier")
    scopes: List[str] = Field(..., description="Requested permissions")
    roots: List[str] = Field(..., description="Allowed file system roots")
    reason: str = Field(..., description="Justification for access")

    # Auto-populated fields
    request_id: Optional[str] = None
    status: RequestStatus = RequestStatus.PENDING
    created_at: Optional[datetime] = None
    approved_at: Optional[datetime] = None
    denied_at: Optional[datetime] = None
    denial_reason: Optional[str] = None
    session_id: Optional[str] = None

    @validator('agent_id')
    def validate_agent_id(cls, v):
        if not re.match(r'^[a-zA-Z0-9\-_]+$', v):
            raise ValueError('Invalid agent_id format')
        if len(v) > 64:
            raise ValueError('agent_id too long')
        return v

    @validator('reason')
    def validate_reason(cls, v):
        if len(v) < 10:
            raise ValueError('Reason too short')
        if len(v) > 500:
            raise ValueError('Reason too long')
        return v

class AccessResponse(BaseModel):
    """Response to access request"""
    request_id: str
    status: str
    created_at: datetime

class SessionInfo(BaseModel):
    """Active session information"""
    session_id: str
    request_id: str
    agent_id: str
    session_token: str
    approved_scopes: List[str]
    allowed_roots: List[str]
    status: str
    created_at: datetime
    expires_at: datetime
    last_activity: datetime
    revoked_at: Optional[datetime] = None
    revocation_reason: Optional[str] = None

class WebSocketMessage(BaseModel):
    """Base WebSocket message"""
    type: str
    id: str

    # Command-specific fields
    action: Optional[str] = None
    args: Optional[Dict[str, Any]] = None

    # Result-specific fields
    status: Optional[str] = None
    data: Optional[Any] = None
    error: Optional[str] = None

    # Event-specific fields
    event: Optional[str] = None

class CommandMessage(BaseModel):
    """Command message from agent to plugin"""
    type: str = "cmd"
    id: str
    action: str
    args: Dict[str, Any] = {}

class ResultMessage(BaseModel):
    """Result message from plugin to agent"""
    type: str = "result"
    id: str
    status: str  # "ok" or "error"
    data: Optional[Any] = None
    error: Optional[str] = None

class EventMessage(BaseModel):
    """Event message from plugin to agent"""
    type: str = "event"
    event: str
    data: Dict[str, Any] = {}
```

---

## 🔍 Audit Logger (`bridge/audit.py`)

```python
"""
Audit Logger - Security and activity logging
"""

import json
import logging
from datetime import datetime
from typing import Any, Dict, Optional

class AuditLogger:
    def __init__(self, logger_name: str = "audit"):
        self.logger = logging.getLogger(logger_name)

        # Setup file handler for audit logs
        handler = logging.FileHandler("logs/audit.log")
        formatter = logging.Formatter(
            '%(asctime)s - AUDIT - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

    def _log_event(self, event_type: str, data: Dict[str, Any]):
        """Log audit event"""
        audit_entry = {
            "event_type": event_type,
            "timestamp": datetime.utcnow().isoformat(),
            **data
        }
        self.logger.info(json.dumps(audit_entry))

    def log_access_request(self, agent_id: str, scopes: list, result: str):
        """Log access request"""
        self._log_event("access_request", {
            "agent_id": agent_id,
            "scopes": scopes,
            "result": result
        })

    def log_session_created(self, session_id: str, agent_id: str, scopes: list):
        """Log session creation"""
        self._log_event("session_created", {
            "session_id": session_id,
            "agent_id": agent_id,
            "scopes": scopes
        })

    def log_session_revoked(self, session_id: str, reason: str):
        """Log session revocation"""
        self._log_event("session_revoked", {
            "session_id": session_id,
            "reason": reason
        })

    def log_command(self, session_id: str, action: str, result: str):
        """Log command execution"""
        self._log_event("command_executed", {
            "session_id": session_id,
            "action": action,
            "result": result
        })

    def log_security_event(self, event: str, details: str):
        """Log security event"""
        self._log_event("security_event", {
            "event": event,
            "details": details
        })

    def log_request_denied(self, request_id: str, reason: str):
        """Log request denial"""
        self._log_event("request_denied", {
            "request_id": request_id,
            "reason": reason
        })
```

---

## 🚦 Rate Limiter (`bridge/rate_limiter.py`)

```python
"""
Rate Limiter - Request rate limiting implementation
"""

import asyncio
import time
from collections import defaultdict
from typing import Dict, List

class RateLimiter:
    def __init__(self, requests_per_window: int, window_seconds: int):
        self.requests_per_window = requests_per_window
        self.window_seconds = window_seconds
        self.requests: Dict[str, List[float]] = defaultdict(list)
        self.total_requests = 0
        self.rejected_requests = 0

    async def is_allowed(self, identifier: str) -> bool:
        """Check if request is allowed under rate limit"""
        now = time.time()
        window_start = now - self.window_seconds

        # Clean old requests
        self.requests[identifier] = [
            req_time for req_time in self.requests[identifier]
            if req_time > window_start
        ]

        self.total_requests += 1

        # Check if limit exceeded
        if len(self.requests[identifier]) >= self.requests_per_window:
            self.rejected_requests += 1
            return False

        # Add current request
        self.requests[identifier].append(now)
        return True

    async def cleanup_task(self):
        """Background cleanup task"""
        while True:
            await asyncio.sleep(self.window_seconds)
            await self._cleanup_old_requests()

    async def _cleanup_old_requests(self):
        """Remove old request records"""
        now = time.time()
        window_start = now - self.window_seconds

        for identifier in list(self.requests.keys()):
            self.requests[identifier] = [
                req_time for req_time in self.requests[identifier]
                if req_time > window_start
            ]

            # Remove empty entries
            if not self.requests[identifier]:
                del self.requests[identifier]
```

---

**Next**: See [TESTING.md](TESTING.md) for comprehensive testing strategies and test cases
