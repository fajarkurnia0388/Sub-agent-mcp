# MCP Local Provider — Implementation Guide

> **Language**: [🇺🇸 English](IMPLEMENTATION.md) | [🇮🇩 Bahasa Indonesia](docs_id/IMPLEMENTATION.md)

> **Complete code examples, starter templates, and implementation details**

## 🎯 Implementation Overview

This document provides complete, production-ready code examples for all components of the MCP Local Provider system. Each component is designed to be modular, testable, and secure.

---

## 📁 Project Structure

```
local_provider/
├── src/
│   ├── __init__.py
│   ├── adapter_provider.py      # Main FastAPI server
│   ├── mcp_bridge.py           # MCP session management
│   ├── cursor_agent.py         # Agent-side MCP handler
│   ├── models.py               # Pydantic data models
│   ├── security.py             # Security utilities
│   ├── monitoring.py           # Health & metrics
│   └── utils.py                # Common utilities
├── config/
│   ├── .env.example
│   ├── security.yaml
│   └── logging.yaml
├── tests/
└── requirements.txt
```

---

## 🚀 Main Components Implementation

### 1. Provider Adapter (FastAPI Server)

#### `src/adapter_provider.py`

```python
"""
MCP Local Provider Adapter - Main FastAPI Application
Exposes OpenAI-compatible API for VoidEditor
"""

import asyncio
import json
import time
import uuid
from datetime import datetime, timedelta
from typing import Dict, List, Optional, AsyncGenerator
import logging

from fastapi import FastAPI, HTTPException, Request, Response
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
import uvicorn

from .models import (
    ChatRequest, ChatResponse, ChatMessage,
    SessionInfo, ModelInvokeRequest, ModelResultResponse
)
from .mcp_bridge import MCPBridge
from .security import SecurityManager, validate_request
from .monitoring import MetricsCollector, HealthMonitor
from .utils import setup_logging, load_config

# Configure logging
logger = setup_logging(__name__)

# Load configuration
config = load_config()

# Initialize FastAPI app
app = FastAPI(
    title="MCP Local Provider Adapter",
    description="Agent-as-Local-Model Provider for VoidEditor",
    version="1.0.0",
    docs_url="/docs" if config.enable_debug else None,
    redoc_url="/redoc" if config.enable_debug else None
)

# Add CORS middleware (localhost only)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://127.0.0.1:*",
        "http://localhost:*"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

# Initialize components
mcp_bridge = MCPBridge(config.mcp_bridge_url, config.mcp_bridge_token)
security_manager = SecurityManager(config)
metrics_collector = MetricsCollector()
health_monitor = HealthMonitor(mcp_bridge, metrics_collector)

# Session storage
active_sessions: Dict[str, SessionInfo] = {}

@app.middleware("http")
async def security_middleware(request: Request, call_next):
    """Apply security checks to all requests"""
    try:
        # Validate request security
        await security_manager.validate_request(request)

        # Record request start time
        start_time = time.time()

        # Process request
        response = await call_next(request)

        # Record metrics
        processing_time = (time.time() - start_time) * 1000
        metrics_collector.record_request(
            endpoint=request.url.path,
            method=request.method,
            status_code=response.status_code,
            processing_time_ms=processing_time
        )

        return response

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Security middleware error: {e}")
        raise HTTPException(500, "Internal security error")

@app.on_event("startup")
async def startup_event():
    """Initialize application on startup"""
    logger.info("Starting MCP Local Provider Adapter")

    # Initialize MCP Bridge connection
    await mcp_bridge.connect()

    # Start background tasks
    asyncio.create_task(cleanup_expired_sessions())
    asyncio.create_task(health_monitor.start_monitoring())

    logger.info(f"Adapter started on {config.host}:{config.port}")

@app.on_event("shutdown")
async def shutdown_event():
    """Cleanup on application shutdown"""
    logger.info("Shutting down MCP Local Provider Adapter")

    # Close MCP Bridge connection
    await mcp_bridge.disconnect()

    # Cleanup sessions
    active_sessions.clear()

    logger.info("Adapter shutdown complete")

# Health check endpoint
@app.get("/health")
async def health_check():
    """Basic health check"""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0"
    }

@app.get("/health/deep")
async def deep_health_check():
    """Detailed health information"""
    health_status = await health_monitor.get_health_status()
    return health_status.dict()

# OpenAI-compatible chat completions endpoint
@app.post("/v1/chat/completions")
async def chat_completions(request: ChatRequest, http_request: Request):
    """
    OpenAI-compatible chat completions endpoint
    Forwards requests to Cursor Agent via MCP Bridge
    """
    try:
        # Get session (required for processing)
        session = await get_active_session(http_request)
        if not session:
            raise HTTPException(
                status_code=503,
                detail="No active agent session available"
            )

        # Validate request against session scope
        await validate_request_scope(request, session)

        # Create MCP invoke request
        invoke_request = ModelInvokeRequest(
            id=str(uuid.uuid4()),
            session_id=session.session_id,
            payload={
                "kind": "chat",
                "messages": [msg.dict() for msg in request.messages],
                "parameters": {
                    "max_tokens": request.max_tokens,
                    "temperature": request.temperature,
                    "stream": request.stream,
                    "top_p": getattr(request, 'top_p', 1.0),
                }
            }
        )

        # Handle streaming vs non-streaming
        if request.stream:
            return StreamingResponse(
                stream_chat_completion(invoke_request, session),
                media_type="text/event-stream"
            )
        else:
            # Send request to agent via MCP Bridge
            result = await mcp_bridge.send_invoke_request(invoke_request)

            # Convert MCP response to OpenAI format
            openai_response = convert_mcp_to_openai_response(result, invoke_request.id)

            # Update session activity
            session.last_activity = datetime.utcnow()
            session.request_count += 1

            return openai_response

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Chat completion error: {e}")
        raise HTTPException(500, "Internal server error")

async def stream_chat_completion(
    invoke_request: ModelInvokeRequest,
    session: SessionInfo
) -> AsyncGenerator[str, None]:
    """Stream chat completion responses"""
    try:
        # Send streaming request to agent
        async for chunk in mcp_bridge.stream_invoke_request(invoke_request):
            # Convert MCP chunk to OpenAI stream format
            openai_chunk = convert_mcp_chunk_to_openai(chunk, invoke_request.id)

            # Yield as Server-Sent Event
            yield f"data: {json.dumps(openai_chunk)}\n\n"

        # Send final chunk
        yield "data: [DONE]\n\n"

        # Update session activity
        session.last_activity = datetime.utcnow()
        session.request_count += 1

    except Exception as e:
        logger.error(f"Streaming error: {e}")
        error_chunk = {
            "id": invoke_request.id,
            "object": "chat.completion.chunk",
            "created": int(time.time()),
            "model": "cursor-agent",
            "choices": [{
                "index": 0,
                "delta": {},
                "finish_reason": "error"
            }]
        }
        yield f"data: {json.dumps(error_chunk)}\n\n"
        yield "data: [DONE]\n\n"

# Control endpoint for session registration
@app.post("/control/register")
async def register_session(
    session_data: dict,
    request: Request
):
    """
    Register agent session (called by MCP Bridge after approval)
    Requires bridge authentication token
    """
    try:
        # Validate bridge authentication
        auth_header = request.headers.get("authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise HTTPException(401, "Missing or invalid authorization")

        token = auth_header.split(" ")[1]
        if not security_manager.verify_bridge_token(token):
            raise HTTPException(401, "Invalid bridge token")

        # Extract session information
        session_id = session_data.get("session_id")
        session_token = session_data.get("session_token")
        agent_id = session_data.get("agent_id")
        allowed_scopes = session_data.get("allowed_scopes", [])
        ttl_seconds = session_data.get("ttl_seconds", 300)
        label = session_data.get("label", "Cursor Agent")

        if not all([session_id, session_token, agent_id]):
            raise HTTPException(400, "Missing required session data")

        # Create session info
        expires_at = datetime.utcnow() + timedelta(seconds=ttl_seconds)
        session = SessionInfo(
            session_id=session_id,
            agent_id=agent_id,
            session_token_hash=security_manager.hash_token(session_token),
            allowed_scopes=allowed_scopes,
            created_at=datetime.utcnow(),
            expires_at=expires_at,
            last_activity=datetime.utcnow(),
            request_count=0,
            label=label,
            status="active"
        )

        # Store session
        active_sessions[session_id] = session

        logger.info(f"Registered session {session_id} for agent {agent_id}")

        return {
            "registered": session_id,
            "status": "active",
            "expires_at": expires_at.isoformat()
        }

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Session registration error: {e}")
        raise HTTPException(500, "Session registration failed")

@app.post("/control/revoke")
async def revoke_session(
    revoke_data: dict,
    request: Request
):
    """Revoke agent session"""
    try:
        # Validate bridge authentication
        auth_header = request.headers.get("authorization")
        if not auth_header or not security_manager.verify_bridge_token(
            auth_header.split(" ")[1] if auth_header.startswith("Bearer ") else ""
        ):
            raise HTTPException(401, "Invalid bridge token")

        session_id = revoke_data.get("session_id")
        if not session_id:
            raise HTTPException(400, "Missing session_id")

        # Remove session
        if session_id in active_sessions:
            del active_sessions[session_id]
            logger.info(f"Revoked session {session_id}")

        return {"revoked": session_id, "status": "success"}

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Session revocation error: {e}")
        raise HTTPException(500, "Session revocation failed")

@app.get("/control/sessions")
async def list_sessions(request: Request):
    """List active sessions (requires bridge authentication)"""
    try:
        # Validate bridge authentication
        auth_header = request.headers.get("authorization")
        if not auth_header or not security_manager.verify_bridge_token(
            auth_header.split(" ")[1] if auth_header.startswith("Bearer ") else ""
        ):
            raise HTTPException(401, "Invalid bridge token")

        sessions_data = []
        for session in active_sessions.values():
            sessions_data.append({
                "session_id": session.session_id,
                "agent_id": session.agent_id,
                "status": session.status,
                "created_at": session.created_at.isoformat(),
                "expires_at": session.expires_at.isoformat(),
                "last_activity": session.last_activity.isoformat(),
                "request_count": session.request_count,
                "label": session.label
            })

        return {
            "sessions": sessions_data,
            "total_count": len(sessions_data),
            "active_count": len([s for s in sessions_data if s["status"] == "active"])
        }

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"List sessions error: {e}")
        raise HTTPException(500, "Failed to list sessions")

# Ollama compatibility endpoints
@app.post("/api/generate")
async def ollama_generate(request: dict, http_request: Request):
    """Ollama-compatible generation endpoint"""
    try:
        # Convert Ollama request to OpenAI format
        openai_request = ChatRequest(
            model=request.get("model", "cursor-agent"),
            messages=[
                ChatMessage(role="user", content=request.get("prompt", ""))
            ],
            temperature=request.get("options", {}).get("temperature", 0.8),
            max_tokens=request.get("options", {}).get("max_tokens", 512),
            stream=request.get("stream", False)
        )

        # Process as chat completion
        if openai_request.stream:
            return StreamingResponse(
                ollama_stream_wrapper(openai_request, http_request),
                media_type="application/x-ndjson"
            )
        else:
            result = await chat_completions(openai_request, http_request)

            # Convert to Ollama format
            ollama_response = {
                "model": result["model"],
                "created_at": datetime.utcnow().isoformat(),
                "response": result["choices"][0]["message"]["content"],
                "done": True,
                "total_duration": 1000000000,  # Mock timing
                "load_duration": 100000000,
                "prompt_eval_count": result["usage"]["prompt_tokens"],
                "eval_count": result["usage"]["completion_tokens"]
            }

            return ollama_response

    except Exception as e:
        logger.error(f"Ollama generate error: {e}")
        raise HTTPException(500, "Generation failed")

async def ollama_stream_wrapper(
    request: ChatRequest,
    http_request: Request
) -> AsyncGenerator[str, None]:
    """Convert OpenAI stream to Ollama format"""
    session = await get_active_session(http_request)
    if not session:
        raise HTTPException(503, "No active agent session")

    invoke_request = ModelInvokeRequest(
        id=str(uuid.uuid4()),
        session_id=session.session_id,
        payload={
            "kind": "chat",
            "messages": [msg.dict() for msg in request.messages],
            "parameters": {
                "max_tokens": request.max_tokens,
                "temperature": request.temperature,
                "stream": True
            }
        }
    )

    async for chunk in mcp_bridge.stream_invoke_request(invoke_request):
        ollama_chunk = {
            "model": "cursor-agent",
            "created_at": datetime.utcnow().isoformat(),
            "response": chunk.get("delta", {}).get("content", ""),
            "done": chunk.get("finish_reason") is not None
        }
        yield json.dumps(ollama_chunk) + "\n"

# Model listing endpoints
@app.get("/v1/models")
async def list_models():
    """OpenAI-compatible model listing"""
    return {
        "object": "list",
        "data": [
            {
                "id": "cursor-agent",
                "object": "model",
                "created": 1677610602,
                "owned_by": "mcp-local-provider",
                "permission": [],
                "root": "cursor-agent",
                "parent": None
            }
        ]
    }

@app.get("/api/tags")
async def ollama_tags():
    """Ollama-compatible model listing"""
    return {
        "models": [
            {
                "name": "cursor-agent:latest",
                "modified_at": datetime.utcnow().isoformat(),
                "size": 0,
                "digest": "sha256:mcp-proxy-agent",
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

# Utility functions
async def get_active_session(request: Request) -> Optional[SessionInfo]:
    """Get active session for request"""
    # Check for session hint in headers
    session_hint = request.headers.get("x-mcp-session")

    if session_hint and session_hint in active_sessions:
        session = active_sessions[session_hint]
        if session.is_valid():
            return session

    # Find any active session
    for session in active_sessions.values():
        if session.is_valid():
            return session

    return None

async def validate_request_scope(request: ChatRequest, session: SessionInfo):
    """Validate request against session scope"""
    if "inference" not in session.allowed_scopes:
        raise HTTPException(403, "Inference not allowed for this session")

    # Check rate limits
    if not await security_manager.check_rate_limit(session.session_id):
        raise HTTPException(429, "Rate limit exceeded")

def convert_mcp_to_openai_response(mcp_result: dict, request_id: str) -> dict:
    """Convert MCP response to OpenAI format"""
    return {
        "id": request_id,
        "object": "chat.completion",
        "created": int(time.time()),
        "model": "cursor-agent",
        "choices": mcp_result.get("result", {}).get("choices", []),
        "usage": mcp_result.get("result", {}).get("usage", {
            "prompt_tokens": 0,
            "completion_tokens": 0,
            "total_tokens": 0
        })
    }

def convert_mcp_chunk_to_openai(mcp_chunk: dict, request_id: str) -> dict:
    """Convert MCP stream chunk to OpenAI format"""
    return {
        "id": request_id,
        "object": "chat.completion.chunk",
        "created": int(time.time()),
        "model": "cursor-agent",
        "choices": [{
            "index": 0,
            "delta": mcp_chunk.get("delta", {}),
            "finish_reason": mcp_chunk.get("finish_reason")
        }]
    }

async def cleanup_expired_sessions():
    """Background task to cleanup expired sessions"""
    while True:
        try:
            current_time = datetime.utcnow()
            expired_sessions = [
                session_id for session_id, session in active_sessions.items()
                if session.expires_at < current_time
            ]

            for session_id in expired_sessions:
                del active_sessions[session_id]
                logger.info(f"Cleaned up expired session: {session_id}")

            # Sleep for 60 seconds before next cleanup
            await asyncio.sleep(60)

        except Exception as e:
            logger.error(f"Session cleanup error: {e}")
            await asyncio.sleep(60)

if __name__ == "__main__":
    uvicorn.run(
        "adapter_provider:app",
        host=config.host,
        port=config.port,
        workers=config.workers,
        log_level=config.log_level.lower(),
        access_log=True
    )
```

### 2. Data Models

#### `src/models.py`

```python
"""
Pydantic data models for MCP Local Provider
"""

from datetime import datetime
from typing import List, Optional, Dict, Any, Literal
from pydantic import BaseModel, Field
from enum import Enum

# Chat API Models
class ChatMessage(BaseModel):
    role: Literal["system", "user", "assistant"]
    content: str
    name: Optional[str] = None

class ChatRequest(BaseModel):
    model: str = Field(default="cursor-agent", regex=r"^[a-zA-Z0-9\-_]+$")
    messages: List[ChatMessage] = Field(min_items=1, max_items=100)
    temperature: float = Field(ge=0.0, le=2.0, default=0.8)
    max_tokens: int = Field(ge=1, le=8192, default=512)
    stream: bool = Field(default=False)
    top_p: float = Field(ge=0.0, le=1.0, default=1.0)
    frequency_penalty: float = Field(ge=-2.0, le=2.0, default=0.0)
    presence_penalty: float = Field(ge=-2.0, le=2.0, default=0.0)

class CompletionChoice(BaseModel):
    index: int
    message: ChatMessage
    finish_reason: Literal["stop", "length", "content_filter"]

class TokenUsage(BaseModel):
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int

class ChatResponse(BaseModel):
    id: str
    object: str = "chat.completion"
    created: int
    model: str
    choices: List[CompletionChoice]
    usage: TokenUsage

# Session Management Models
class SessionStatus(str, Enum):
    ACTIVE = "active"
    EXPIRED = "expired"
    REVOKED = "revoked"
    SUSPENDED = "suspended"

class SessionInfo(BaseModel):
    session_id: str
    agent_id: str
    session_token_hash: str  # Never store plain tokens
    allowed_scopes: List[str]
    created_at: datetime
    expires_at: datetime
    last_activity: datetime
    request_count: int
    max_requests: int = 100
    label: str
    status: SessionStatus = SessionStatus.ACTIVE

    def is_valid(self) -> bool:
        """Check if session is still valid"""
        now = datetime.utcnow()
        return (
            self.status == SessionStatus.ACTIVE and
            now < self.expires_at and
            self.request_count < self.max_requests
        )

# MCP Protocol Models
class ModelMetadata(BaseModel):
    provider: str = "mcp-proxy"
    label: str = "cursor-agent-as-model"
    requested_scopes: List[str] = ["inference"]

class InferencePayload(BaseModel):
    kind: Literal["chat", "completion", "embedding"]
    messages: List[Dict[str, Any]]
    parameters: Dict[str, Any]

class ModelInvokeRequest(BaseModel):
    type: str = "model_invoke"
    id: str
    session_id: str
    model_meta: ModelMetadata = ModelMetadata()
    payload: InferencePayload

class ModelResultResponse(BaseModel):
    type: str = "model_result"
    id: str
    status: Literal["ok", "error", "timeout"]
    result: Optional[Dict[str, Any]] = None
    error: Optional[Dict[str, Any]] = None

class ModelStreamChunk(BaseModel):
    type: str = "model_stream_chunk"
    id: str
    chunk_index: int
    delta: Dict[str, Any]
    finish_reason: Optional[str] = None

# Health & Monitoring Models
class ComponentHealth(BaseModel):
    name: str
    status: Literal["healthy", "degraded", "unhealthy"]
    last_check: datetime
    error_count: int = 0
    latency_ms: float = 0.0
    details: Dict[str, Any] = {}

class SystemMetrics(BaseModel):
    # Request metrics
    total_requests: int = 0
    requests_per_second: float = 0.0
    average_latency_ms: float = 0.0
    error_rate: float = 0.0

    # Session metrics
    active_sessions: int = 0
    session_creation_rate: float = 0.0
    session_expiry_rate: float = 0.0

    # System metrics
    cpu_usage_percent: float = 0.0
    memory_usage_mb: float = 0.0
    disk_usage_mb: float = 0.0

    # Application metrics
    connection_pool_size: int = 0
    queue_depth: int = 0

class HealthStatus(BaseModel):
    status: Literal["healthy", "degraded", "unhealthy"]
    timestamp: datetime
    components: Dict[str, ComponentHealth]
    metrics: SystemMetrics

# Configuration Models
class AdapterConfig(BaseModel):
    # Server settings
    host: str = "127.0.0.1"
    port: int = 11434
    workers: int = 1

    # MCP Bridge settings
    mcp_bridge_url: str = "ws://127.0.0.1:8787"
    mcp_bridge_token: str = "dev-token"

    # Security settings
    session_ttl_seconds: int = 300
    max_requests_per_session: int = 100
    max_token_length: int = 8192

    # Feature flags
    enable_streaming: bool = True
    enable_ollama_compat: bool = True
    enable_debug: bool = False

    # Logging
    log_level: str = "INFO"
    audit_log_file: str = "./logs/audit.jsonl"

# Audit Models
class AuditLogEntry(BaseModel):
    timestamp: datetime
    event_type: Literal["REQUEST", "RESPONSE", "ERROR", "SECURITY", "SESSION"]
    session_id: Optional[str] = None
    client_ip: str
    user_agent: str = ""
    request_id: str
    endpoint: str
    method: str
    status_code: int
    processing_time_ms: float
    request_size_bytes: int = 0
    response_size_bytes: int = 0
    prompt_hash: Optional[str] = None  # SHA256 of prompt
    error_details: Optional[str] = None
    security_flags: List[str] = []
```

### 3. MCP Bridge Integration

#### `src/mcp_bridge.py`

```python
"""
MCP Bridge - Session management and Agent communication
"""

import asyncio
import json
import logging
import time
import websockets
from typing import Optional, Dict, AsyncGenerator, Any
from datetime import datetime

from .models import ModelInvokeRequest, ModelResultResponse, ModelStreamChunk
from .utils import setup_logging

logger = setup_logging(__name__)

class MCPBridge:
    """MCP Bridge for communicating with Cursor Agent"""

    def __init__(self, bridge_url: str, bridge_token: str):
        self.bridge_url = bridge_url
        self.bridge_token = bridge_token
        self.websocket: Optional[websockets.WebSocketServerProtocol] = None
        self.connected = False
        self.pending_requests: Dict[str, asyncio.Future] = {}
        self.streaming_requests: Dict[str, asyncio.Queue] = {}

    async def connect(self):
        """Connect to MCP Bridge WebSocket"""
        try:
            headers = {"Authorization": f"Bearer {self.bridge_token}"}

            self.websocket = await websockets.connect(
                self.bridge_url,
                extra_headers=headers,
                ping_interval=30,
                ping_timeout=10
            )

            self.connected = True
            logger.info(f"Connected to MCP Bridge: {self.bridge_url}")

            # Start message handling task
            asyncio.create_task(self._handle_messages())

        except Exception as e:
            logger.error(f"Failed to connect to MCP Bridge: {e}")
            raise

    async def disconnect(self):
        """Disconnect from MCP Bridge"""
        self.connected = False

        if self.websocket:
            await self.websocket.close()
            self.websocket = None

        # Cancel pending requests
        for future in self.pending_requests.values():
            if not future.done():
                future.cancel()
        self.pending_requests.clear()

        # Clear streaming requests
        self.streaming_requests.clear()

        logger.info("Disconnected from MCP Bridge")

    async def send_invoke_request(
        self,
        request: ModelInvokeRequest,
        timeout: int = 30
    ) -> Dict[str, Any]:
        """Send non-streaming invoke request to agent"""
        if not self.connected or not self.websocket:
            raise Exception("Not connected to MCP Bridge")

        # Create future for response
        future = asyncio.Future()
        self.pending_requests[request.id] = future

        try:
            # Send request
            await self.websocket.send(request.json())

            # Wait for response
            result = await asyncio.wait_for(future, timeout=timeout)
            return result

        except asyncio.TimeoutError:
            logger.error(f"Request {request.id} timed out after {timeout}s")
            raise Exception(f"Agent response timeout after {timeout} seconds")

        except Exception as e:
            logger.error(f"Request {request.id} failed: {e}")
            raise

        finally:
            # Cleanup
            self.pending_requests.pop(request.id, None)

    async def stream_invoke_request(
        self,
        request: ModelInvokeRequest
    ) -> AsyncGenerator[Dict[str, Any], None]:
        """Send streaming invoke request to agent"""
        if not self.connected or not self.websocket:
            raise Exception("Not connected to MCP Bridge")

        # Create queue for streaming chunks
        queue = asyncio.Queue()
        self.streaming_requests[request.id] = queue

        try:
            # Send streaming request
            request.payload["parameters"]["stream"] = True
            await self.websocket.send(request.json())

            # Yield chunks as they arrive
            while True:
                chunk = await queue.get()

                # Check for end marker
                if chunk.get("finish_reason") is not None:
                    yield chunk
                    break

                yield chunk

        except Exception as e:
            logger.error(f"Streaming request {request.id} failed: {e}")
            raise

        finally:
            # Cleanup
            self.streaming_requests.pop(request.id, None)

    async def _handle_messages(self):
        """Handle incoming messages from MCP Bridge"""
        try:
            async for message in self.websocket:
                try:
                    data = json.loads(message)
                    await self._process_message(data)

                except json.JSONDecodeError as e:
                    logger.error(f"Invalid JSON from bridge: {e}")

                except Exception as e:
                    logger.error(f"Message processing error: {e}")

        except websockets.exceptions.ConnectionClosed:
            logger.warning("MCP Bridge connection closed")
            self.connected = False

        except Exception as e:
            logger.error(f"Message handling error: {e}")
            self.connected = False

    async def _process_message(self, data: Dict[str, Any]):
        """Process incoming message from agent"""
        message_type = data.get("type")
        message_id = data.get("id")

        if message_type == "model_result":
            # Handle non-streaming response
            future = self.pending_requests.get(message_id)
            if future and not future.done():
                future.set_result(data)
            else:
                logger.warning(f"Received unexpected result for {message_id}")

        elif message_type == "model_stream_chunk":
            # Handle streaming chunk
            queue = self.streaming_requests.get(message_id)
            if queue:
                await queue.put(data)
            else:
                logger.warning(f"Received unexpected chunk for {message_id}")

        elif message_type == "error":
            # Handle error response
            future = self.pending_requests.get(message_id)
            if future and not future.done():
                error_msg = data.get("error", {}).get("message", "Unknown error")
                future.set_exception(Exception(error_msg))

        else:
            logger.warning(f"Unknown message type: {message_type}")

    async def ping(self) -> bool:
        """Check connection health"""
        try:
            if not self.connected or not self.websocket:
                return False

            pong_waiter = await self.websocket.ping()
            await asyncio.wait_for(pong_waiter, timeout=5.0)
            return True

        except Exception:
            return False

    async def get_connection_status(self) -> Dict[str, Any]:
        """Get detailed connection status"""
        return {
            "connected": self.connected,
            "websocket_open": self.websocket is not None and not self.websocket.closed,
            "bridge_url": self.bridge_url,
            "pending_requests": len(self.pending_requests),
            "streaming_requests": len(self.streaming_requests),
            "last_ping": await self.ping()
        }

class MCPBridgeManager:
    """Manager for multiple MCP Bridge connections"""

    def __init__(self):
        self.bridges: Dict[str, MCPBridge] = {}
        self.default_bridge: Optional[MCPBridge] = None

    def add_bridge(self, name: str, bridge: MCPBridge, is_default: bool = False):
        """Add a bridge to the manager"""
        self.bridges[name] = bridge
        if is_default or not self.default_bridge:
            self.default_bridge = bridge

    async def connect_all(self):
        """Connect all bridges"""
        for name, bridge in self.bridges.items():
            try:
                await bridge.connect()
                logger.info(f"Connected bridge: {name}")
            except Exception as e:
                logger.error(f"Failed to connect bridge {name}: {e}")

    async def disconnect_all(self):
        """Disconnect all bridges"""
        for name, bridge in self.bridges.items():
            try:
                await bridge.disconnect()
                logger.info(f"Disconnected bridge: {name}")
            except Exception as e:
                logger.error(f"Failed to disconnect bridge {name}: {e}")

    def get_bridge(self, name: Optional[str] = None) -> Optional[MCPBridge]:
        """Get a bridge by name or default"""
        if name:
            return self.bridges.get(name)
        return self.default_bridge

    async def health_check_all(self) -> Dict[str, Dict[str, Any]]:
        """Health check all bridges"""
        results = {}
        for name, bridge in self.bridges.items():
            results[name] = await bridge.get_connection_status()
        return results
```

---

**Next**: See [TESTING.md](TESTING.md) for comprehensive testing strategies and validation
