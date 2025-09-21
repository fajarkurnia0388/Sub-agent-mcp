# MCP Local Provider — Panduan Implementasi

> **Bahasa**: [🇺🇸 English](../IMPLEMENTATION.md) | [🇮🇩 Bahasa Indonesia](docs_id/IMPLEMENTATION.md)

**Contoh kode lengkap dan detail implementasi untuk sistem MCP Local Provider**

## 🏗️ Implementasi Arsitektur

MCP Local Provider diimplementasikan sebagai aplikasi FastAPI dengan dukungan WebSocket, menyediakan komunikasi aman antara VoidEditor dan agen Cursor.

### Komponen Inti

1. **FastAPI Server** - HTTP/WebSocket endpoints
2. **Session Manager** - Manajemen sesi efemeral
3. **MCP Client** - Komunikasi dengan agen Cursor
4. **Security Layer** - Autentikasi & otorisasi
5. **Rate Limiter** - Pembatasan request

---

## 🚀 Aplikasi Utama (`local_provider/adapter.py`)

```python
#!/usr/bin/env python3
"""
MCP Local Provider - Provider Adapter
Menyediakan API kompatibel OpenAI/Ollama untuk agen Cursor
"""

import asyncio
import uvicorn
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
import logging
import os
from typing import List, Optional, AsyncGenerator
import yaml
from datetime import datetime
import json

from .session_manager import SessionManager
from .mcp_client import MCPClient
from .models import (
    ChatCompletionRequest, ChatCompletionResponse,
    ModelListResponse, HealthResponse
)
from .security import SecurityManager
from .rate_limiter import RateLimiter
from .audit import AuditLogger

# Load configuration
def load_config():
    config_path = os.getenv("LOCAL_PROVIDER_CONFIG", "config/config.yaml")
    with open(config_path, 'r') as f:
        return yaml.safe_load(f)

config = load_config()

# Initialize logging
logging.basicConfig(
    level=config['logging']['level'],
    format=config['logging']['format']
)
logger = logging.getLogger(__name__)

# Initialize FastAPI app
app = FastAPI(
    title="MCP Local Provider",
    description="Local AI model provider menggunakan agen Cursor",
    version="1.0.0",
    docs_url="/docs" if config['provider']['debug'] else None
)

# Security setup
security = HTTPBearer()
security_manager = SecurityManager(config['security'])
audit_logger = AuditLogger()
rate_limiter = RateLimiter(
    requests_per_minute=config['rate_limiting']['requests_per_minute'],
    tokens_per_minute=config['rate_limiting']['tokens_per_minute']
)

# Core managers
session_manager = SessionManager(config['sessions'])
mcp_client = MCPClient(config['mcp'])

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=config['security']['allowed_origins'],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

# Authentication dependency
async def verify_api_key(
    request: Request,
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    client_ip = request.client.host
    api_key = credentials.credentials

    # Validate API key
    if not security_manager.verify_api_key(api_key):
        audit_logger.log_authentication(client_ip, api_key[:10], False)
        raise HTTPException(status_code=401, detail="Invalid API key")

    # Check rate limits
    is_allowed, rate_info = await rate_limiter.check_rate_limit(client_ip)
    if not is_allowed:
        audit_logger.log_rate_limit(
            client_ip, rate_info["error"],
            rate_info["current"], rate_info["limit"]
        )
        raise HTTPException(status_code=429, detail=rate_info["error"])

    audit_logger.log_authentication(client_ip, api_key[:10], True)
    return {"client_ip": client_ip, "api_key": api_key}

# ============================================================================
# OpenAI Compatible API
# ============================================================================

@app.post("/v1/chat/completions", response_model=ChatCompletionResponse)
async def chat_completions(
    request: ChatCompletionRequest,
    auth_data: dict = Depends(verify_api_key)
):
    """OpenAI compatible chat completions endpoint"""
    try:
        client_ip = auth_data["client_ip"]

        # Estimate token count untuk rate limiting
        estimated_tokens = estimate_tokens(request.messages, request.max_tokens or 2048)

        # Check token rate limit
        is_allowed, rate_info = await rate_limiter.check_rate_limit(
            client_ip, estimated_tokens
        )
        if not is_allowed:
            raise HTTPException(status_code=429, detail=rate_info["error"])

        # Create session
        session = await session_manager.create_session(
            client_ip=client_ip,
            model=request.model,
            max_tokens=request.max_tokens or 2048
        )

        # Process request
        if request.stream:
            return StreamingResponse(
                stream_chat_completion(session, request),
                media_type="text/event-stream",
                headers={
                    "Cache-Control": "no-cache",
                    "Connection": "keep-alive",
                    "X-Session-ID": session.session_id
                }
            )
        else:
            result = await process_chat_completion(session, request)
            audit_logger.log_model_request(
                client_ip, request.model, estimated_tokens, True
            )
            return result

    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Chat completion error: {e}")
        audit_logger.log_model_request(
            auth_data["client_ip"], request.model, 0, False
        )
        raise HTTPException(status_code=500, detail="Internal server error")

@app.get("/v1/models", response_model=ModelListResponse)
async def list_models(_: dict = Depends(verify_api_key)):
    """List available models"""
    return ModelListResponse(
        object="list",
        data=[
            {
                "id": config['model']['model_name'],
                "object": "model",
                "created": int(datetime.utcnow().timestamp()),
                "owned_by": "cursor-local-provider",
                "permission": [],
                "root": config['model']['model_name'],
                "parent": None
            }
        ]
    )

# ============================================================================
# Ollama Compatible API
# ============================================================================

@app.post("/api/chat")
async def ollama_chat(
    request: dict,
    auth_data: dict = Depends(verify_api_key)
):
    """Ollama compatible chat endpoint"""
    try:
        # Convert Ollama format ke OpenAI format
        openai_request = ChatCompletionRequest(
            model=request.get("model", config['model']['model_name']),
            messages=request.get("messages", []),
            stream=request.get("stream", False),
            max_tokens=request.get("options", {}).get("num_predict", 2048),
            temperature=request.get("options", {}).get("temperature", 0.7)
        )

        # Process sama seperti OpenAI endpoint
        client_ip = auth_data["client_ip"]
        session = await session_manager.create_session(
            client_ip=client_ip,
            model=openai_request.model
        )

        if openai_request.stream:
            return StreamingResponse(
                stream_ollama_chat(session, openai_request),
                media_type="application/json"
            )
        else:
            result = await process_chat_completion(session, openai_request)

            # Convert ke format Ollama
            ollama_response = {
                "model": result.model,
                "created_at": datetime.utcnow().isoformat(),
                "message": {
                    "role": "assistant",
                    "content": result.choices[0].message.content
                },
                "done": True
            }
            return ollama_response

    except Exception as e:
        logger.error(f"Ollama chat error: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.get("/api/tags")
async def ollama_tags(_: dict = Depends(verify_api_key)):
    """List available models (Ollama format)"""
    return {
        "models": [
            {
                "name": config['model']['model_name'],
                "modified_at": datetime.utcnow().isoformat(),
                "size": 0,
                "digest": "sha256:cursor-agent-local",
                "details": {
                    "format": "cursor-agent",
                    "family": "local-provider",
                    "families": ["local-provider"],
                    "parameter_size": "Variable",
                    "quantization_level": "N/A"
                }
            }
        ]
    }

# ============================================================================
# Health & Monitoring Endpoints
# ============================================================================

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint"""
    mcp_status = await mcp_client.get_connection_status()

    return HealthResponse(
        status="healthy",
        timestamp=datetime.utcnow(),
        version="1.0.0",
        uptime_seconds=int(time.time() - start_time),
        active_sessions=len(session_manager.sessions),
        mcp_connection=mcp_status
    )

@app.get("/metrics")
async def get_metrics(_: dict = Depends(verify_api_key)):
    """Performance metrics"""
    return {
        "sessions": {
            "active": len(session_manager.sessions),
            "total_created": session_manager.total_created,
            "total_expired": session_manager.total_expired
        },
        "requests": {
            "total": rate_limiter.total_requests,
            "successful": rate_limiter.successful_requests,
            "failed": rate_limiter.failed_requests,
            "rate_limited": rate_limiter.rate_limited_requests
        },
        "performance": {
            "avg_response_time_ms": performance_monitor.avg_response_time,
            "memory_usage_mb": psutil.Process().memory_info().rss / 1024 / 1024
        },
        "mcp": {
            "connection_status": await mcp_client.get_connection_status(),
            "messages_sent": mcp_client.messages_sent,
            "messages_received": mcp_client.messages_received
        }
    }

# ============================================================================
# Processing Functions
# ============================================================================

async def process_chat_completion(
    session: Session,
    request: ChatCompletionRequest
) -> ChatCompletionResponse:
    """Process non-streaming chat completion"""

    # Send request ke agen via MCP
    mcp_request = {
        "type": "model_invoke",
        "id": f"req-{session.session_id}",
        "model": request.model,
        "messages": request.messages,
        "stream": False,
        "max_tokens": request.max_tokens,
        "temperature": request.temperature
    }

    response = await mcp_client.send_message(mcp_request)

    if response["type"] == "model_result":
        return ChatCompletionResponse(
            id=f"chatcmpl-{session.session_id}",
            object="chat.completion",
            created=int(datetime.utcnow().timestamp()),
            model=request.model,
            choices=response["choices"],
            usage=response.get("usage", {
                "prompt_tokens": estimate_prompt_tokens(request.messages),
                "completion_tokens": estimate_completion_tokens(response["choices"]),
                "total_tokens": 0
            })
        )
    else:
        raise HTTPException(status_code=500, detail="Agent error")

async def stream_chat_completion(
    session: Session,
    request: ChatCompletionRequest
) -> AsyncGenerator[str, None]:
    """Stream chat completion responses"""

    # Send streaming request ke agen
    mcp_request = {
        "type": "model_invoke",
        "id": f"req-{session.session_id}",
        "model": request.model,
        "messages": request.messages,
        "stream": True,
        "max_tokens": request.max_tokens,
        "temperature": request.temperature
    }

    await mcp_client.send_message(mcp_request)

    # Stream responses
    async for chunk in mcp_client.stream_responses(f"req-{session.session_id}"):
        if chunk["type"] == "model_stream_chunk":
            chunk_data = {
                "id": f"chatcmpl-{session.session_id}",
                "object": "chat.completion.chunk",
                "created": int(datetime.utcnow().timestamp()),
                "model": request.model,
                "choices": [{
                    "index": 0,
                    "delta": chunk["chunk"].get("delta", {}),
                    "finish_reason": chunk["chunk"].get("finish_reason")
                }]
            }

            yield f"data: {json.dumps(chunk_data)}\n\n"

            if chunk.get("finished", False):
                break

    yield "data: [DONE]\n\n"

async def stream_ollama_chat(
    session: Session,
    request: ChatCompletionRequest
) -> AsyncGenerator[str, None]:
    """Stream Ollama format responses"""

    mcp_request = {
        "type": "model_invoke",
        "id": f"req-{session.session_id}",
        "model": request.model,
        "messages": request.messages,
        "stream": True
    }

    await mcp_client.send_message(mcp_request)

    async for chunk in mcp_client.stream_responses(f"req-{session.session_id}"):
        if chunk["type"] == "model_stream_chunk":
            ollama_chunk = {
                "model": request.model,
                "created_at": datetime.utcnow().isoformat(),
                "message": {
                    "role": "assistant",
                    "content": chunk["chunk"].get("delta", {}).get("content", "")
                },
                "done": chunk.get("finished", False)
            }

            yield f"{json.dumps(ollama_chunk)}\n"

            if chunk.get("finished", False):
                break

# ============================================================================
# Utility Functions
# ============================================================================

def estimate_tokens(messages: List[dict], max_tokens: int) -> int:
    """Estimate token count untuk rate limiting"""
    # Simple estimation: ~4 karakter per token
    text_length = sum(len(msg.get("content", "")) for msg in messages)
    estimated_prompt_tokens = text_length // 4
    return estimated_prompt_tokens + max_tokens

def estimate_prompt_tokens(messages: List[dict]) -> int:
    """Estimate prompt tokens"""
    return sum(len(msg.get("content", "")) // 4 for msg in messages)

def estimate_completion_tokens(choices: List[dict]) -> int:
    """Estimate completion tokens"""
    return sum(
        len(choice.get("message", {}).get("content", "")) // 4
        for choice in choices
    )

# ============================================================================
# Startup & Shutdown
# ============================================================================

start_time = time.time()

@app.on_event("startup")
async def startup_event():
    """Initialize komponen saat startup"""
    logger.info("Starting MCP Local Provider")

    # Connect ke MCP bridge
    await mcp_client.connect()

    # Start cleanup tasks
    asyncio.create_task(session_manager.cleanup_task())
    asyncio.create_task(rate_limiter.cleanup_task())

    logger.info("Provider started successfully")

@app.on_event("shutdown")
async def shutdown_event():
    """Cleanup saat shutdown"""
    logger.info("Shutting down MCP Local Provider")

    # Close all sessions
    await session_manager.close_all_sessions()

    # Disconnect MCP client
    await mcp_client.disconnect()

    logger.info("Provider shut down successfully")

# ============================================================================
# Main Entry Point
# ============================================================================

def main():
    """Main entry point"""
    host = config['provider']['host']
    port = config['provider']['port']
    debug = config['provider']['debug']

    logger.info(f"Starting provider pada {host}:{port}")

    uvicorn.run(
        "local_provider.adapter:app",
        host=host,
        port=port,
        reload=debug,
        log_level="info" if not debug else "debug"
    )

if __name__ == "__main__":
    main()
```

---

## 🔐 Security Manager (`local_provider/security.py`)

```python
"""
Security Manager - Autentikasi dan otorisasi
"""

import hashlib
import secrets
from datetime import datetime, timedelta
from typing import Optional
import os

class SecurityManager:
    def __init__(self, config: dict):
        self.config = config
        self.api_key_hash = self._hash_api_key(config['api_key'])

    def _hash_api_key(self, api_key: str) -> str:
        """Hash API key untuk secure comparison"""
        return hashlib.sha256(api_key.encode()).hexdigest()

    def verify_api_key(self, api_key: str) -> bool:
        """Verify API key dengan constant-time comparison"""
        return secrets.compare_digest(
            self._hash_api_key(api_key),
            self.api_key_hash
        )

    def generate_session_id(self) -> str:
        """Generate unique session ID"""
        return f"sess-{secrets.token_urlsafe(32)}"

    def validate_request_data(self, data: dict) -> bool:
        """Validate request data untuk security"""
        # Check required fields
        if not data.get("model"):
            return False

        if not data.get("messages"):
            return False

        # Validate model name
        if data["model"] != "cursor-agent-local":
            return False

        # Validate message format
        for message in data["messages"]:
            if not isinstance(message, dict):
                return False
            if "role" not in message or "content" not in message:
                return False
            if message["role"] not in ["system", "user", "assistant"]:
                return False

        return True

    def sanitize_content(self, content: str) -> str:
        """Sanitize content untuk mencegah injection"""
        # Limit length
        if len(content) > 100000:  # 100KB limit
            content = content[:100000]

        # Remove potentially dangerous patterns
        import re
        content = re.sub(r'<script[^>]*>.*?</script>', '', content, flags=re.IGNORECASE)
        content = re.sub(r'javascript:', '', content, flags=re.IGNORECASE)

        return content
```

---

## 📋 Session Manager (`local_provider/session_manager.py`)

```python
"""
Session Manager - Manajemen sesi efemeral
"""

import asyncio
import time
from datetime import datetime, timedelta
from typing import Dict, Optional
import logging

from .models import Session
from .security import SecurityManager

logger = logging.getLogger(__name__)

class SessionManager:
    def __init__(self, config: dict):
        self.config = config
        self.sessions: Dict[str, Session] = {}
        self.total_created = 0
        self.total_expired = 0

        # Configuration
        self.session_ttl = config.get('session_ttl', 300)
        self.max_sessions = config.get('max_sessions', 10)
        self.cleanup_interval = config.get('cleanup_interval', 60)

        # Security manager
        self.security = SecurityManager({})

    async def create_session(
        self,
        client_ip: str,
        model: str,
        max_tokens: int = 2048
    ) -> Session:
        """Create new session"""

        # Check session limits
        if len(self.sessions) >= self.max_sessions:
            raise ValueError("Maximum sessions reached")

        # Generate session ID
        session_id = self.security.generate_session_id()

        # Create session object
        session = Session(
            session_id=session_id,
            client_ip=client_ip,
            model=model,
            max_tokens=max_tokens,
            created_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + timedelta(seconds=self.session_ttl),
            last_activity=datetime.utcnow()
        )

        # Store session
        self.sessions[session_id] = session
        self.total_created += 1

        logger.info(f"Created session: {session_id} for {client_ip}")
        return session

    async def get_session(self, session_id: str) -> Optional[Session]:
        """Get session by ID"""
        session = self.sessions.get(session_id)

        if not session:
            return None

        # Check expiration
        if datetime.utcnow() > session.expires_at:
            await self.expire_session(session_id)
            return None

        # Update last activity
        session.last_activity = datetime.utcnow()
        return session

    async def expire_session(self, session_id: str):
        """Mark session as expired"""
        if session_id in self.sessions:
            del self.sessions[session_id]
            self.total_expired += 1
            logger.info(f"Session expired: {session_id}")

    async def close_all_sessions(self):
        """Close all active sessions"""
        session_ids = list(self.sessions.keys())
        for session_id in session_ids:
            await self.expire_session(session_id)
        logger.info(f"Closed {len(session_ids)} sessions")

    async def cleanup_task(self):
        """Background task untuk cleanup expired sessions"""
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
            if now > session.expires_at:
                expired_sessions.append(session_id)

        for session_id in expired_sessions:
            await self.expire_session(session_id)

        if expired_sessions:
            logger.info(f"Cleaned up {len(expired_sessions)} expired sessions")
```

---

## 🔌 MCP Client (`local_provider/mcp_client.py`)

```python
"""
MCP Client - Komunikasi dengan agen Cursor
"""

import asyncio
import json
import logging
import websockets
from typing import Dict, AsyncGenerator, Optional
from datetime import datetime
import time

logger = logging.getLogger(__name__)

class MCPClient:
    def __init__(self, config: dict):
        self.config = config
        self.websocket: Optional[websockets.WebSocketServerProtocol] = None
        self.is_connected_flag = False
        self.last_ping_time: Optional[datetime] = None

        # Message tracking
        self.pending_requests: Dict[str, asyncio.Future] = {}
        self.streaming_requests: Dict[str, asyncio.Queue] = {}

        # Statistics
        self.messages_sent = 0
        self.messages_received = 0

        # Configuration
        self.bridge_url = config['bridge_url']
        self.agent_id = config['agent_id']
        self.connection_timeout = config.get('connection_timeout', 30)
        self.heartbeat_interval = config.get('heartbeat_interval', 30)
        self.reconnect_attempts = config.get('reconnect_attempts', 5)
        self.reconnect_delay = config.get('reconnect_delay', 5)

    async def connect(self):
        """Connect ke MCP bridge"""
        for attempt in range(self.reconnect_attempts):
            try:
                logger.info(f"Connecting to MCP bridge: {self.bridge_url}")

                self.websocket = await websockets.connect(
                    self.bridge_url,
                    timeout=self.connection_timeout
                )

                self.is_connected_flag = True
                logger.info("Connected to MCP bridge")

                # Start message handling
                asyncio.create_task(self._handle_messages())
                asyncio.create_task(self._heartbeat_task())

                return

            except Exception as e:
                logger.error(f"Connection attempt {attempt + 1} failed: {e}")
                if attempt < self.reconnect_attempts - 1:
                    await asyncio.sleep(self.reconnect_delay)

        raise ConnectionError("Failed to connect to MCP bridge")

    async def disconnect(self):
        """Disconnect dari MCP bridge"""
        self.is_connected_flag = False

        if self.websocket:
            await self.websocket.close()
            logger.info("Disconnected from MCP bridge")

    async def send_message(self, message: dict) -> dict:
        """Send message dan tunggu response"""
        if not self.is_connected_flag:
            raise ConnectionError("Not connected to MCP bridge")

        message_id = message["id"]

        # Create future untuk response
        future = asyncio.Future()
        self.pending_requests[message_id] = future

        try:
            # Send message
            await self.websocket.send(json.dumps(message))
            self.messages_sent += 1

            # Wait untuk response
            response = await asyncio.wait_for(future, timeout=30.0)
            return response

        except asyncio.TimeoutError:
            raise TimeoutError("Request timeout")
        finally:
            # Cleanup
            self.pending_requests.pop(message_id, None)

    async def stream_responses(self, request_id: str) -> AsyncGenerator[dict, None]:
        """Stream responses untuk streaming requests"""
        if not self.is_connected_flag:
            raise ConnectionError("Not connected to MCP bridge")

        # Create queue untuk streaming responses
        queue = asyncio.Queue()
        self.streaming_requests[request_id] = queue

        try:
            while True:
                # Wait untuk next chunk
                chunk = await asyncio.wait_for(queue.get(), timeout=60.0)

                if chunk is None:  # End of stream
                    break

                yield chunk

        except asyncio.TimeoutError:
            raise TimeoutError("Stream timeout")
        finally:
            # Cleanup
            self.streaming_requests.pop(request_id, None)

    async def _handle_messages(self):
        """Handle incoming messages dari bridge"""
        try:
            async for message_raw in self.websocket:
                try:
                    message = json.loads(message_raw)
                    self.messages_received += 1

                    await self._process_message(message)

                except json.JSONDecodeError as e:
                    logger.error(f"Invalid JSON received: {e}")
                except Exception as e:
                    logger.error(f"Error processing message: {e}")

        except websockets.exceptions.ConnectionClosed:
            logger.warning("WebSocket connection closed")
            self.is_connected_flag = False
        except Exception as e:
            logger.error(f"Message handling error: {e}")
            self.is_connected_flag = False

    async def _process_message(self, message: dict):
        """Process incoming message"""
        message_type = message.get("type")
        message_id = message.get("id")

        if message_type == "model_result":
            # Non-streaming response
            future = self.pending_requests.get(message_id)
            if future and not future.done():
                future.set_result(message)

        elif message_type == "model_stream_chunk":
            # Streaming response
            queue = self.streaming_requests.get(message_id)
            if queue:
                await queue.put(message)

                # Check jika stream finished
                if message.get("finished", False):
                    await queue.put(None)  # End of stream marker

        elif message_type == "error":
            # Error response
            future = self.pending_requests.get(message_id)
            if future and not future.done():
                future.set_exception(Exception(message.get("error", "Unknown error")))

        else:
            logger.warning(f"Unknown message type: {message_type}")

    async def _heartbeat_task(self):
        """Send heartbeat messages"""
        while self.is_connected_flag:
            try:
                if self.websocket:
                    ping_message = {
                        "type": "ping",
                        "timestamp": datetime.utcnow().isoformat()
                    }
                    await self.websocket.send(json.dumps(ping_message))
                    self.last_ping_time = datetime.utcnow()

                await asyncio.sleep(self.heartbeat_interval)

            except Exception as e:
                logger.error(f"Heartbeat error: {e}")
                break

    async def get_connection_status(self) -> dict:
        """Get connection status"""
        return {
            "status": "connected" if self.is_connected_flag else "disconnected",
            "last_ping": self.last_ping_time.isoformat() if self.last_ping_time else None,
            "messages_sent": self.messages_sent,
            "messages_received": self.messages_received,
            "pending_requests": len(self.pending_requests),
            "active_streams": len(self.streaming_requests)
        }

    def is_connected(self) -> bool:
        """Check jika client connected"""
        return self.is_connected_flag
```

---

## 📊 Data Models (`local_provider/models.py`)

```python
"""
Data Models - Pydantic models untuk API
"""

from pydantic import BaseModel, Field, validator
from typing import List, Optional, Dict, Any, Union
from datetime import datetime
from enum import Enum

# ============================================================================
# OpenAI Compatible Models
# ============================================================================

class ChatMessage(BaseModel):
    """Chat message model"""
    role: str = Field(..., description="Peran pengirim: system, user, assistant")
    content: str = Field(..., description="Konten pesan")
    name: Optional[str] = Field(None, description="Nama pengirim (opsional)")

    @validator('role')
    def validate_role(cls, v):
        if v not in ["system", "user", "assistant"]:
            raise ValueError('role harus salah satu dari: system, user, assistant')
        return v

    @validator('content')
    def validate_content(cls, v):
        if len(v) > 100000:  # 100KB limit
            raise ValueError('content terlalu panjang')
        return v

class ChatCompletionRequest(BaseModel):
    """Chat completion request"""
    model: str = Field(..., description="Model identifier")
    messages: List[ChatMessage] = Field(..., description="Array pesan percakapan")
    stream: Optional[bool] = Field(False, description="Stream response")
    max_tokens: Optional[int] = Field(2048, description="Token maksimal")
    temperature: Optional[float] = Field(0.7, description="Sampling temperature")
    top_p: Optional[float] = Field(1.0, description="Nucleus sampling")
    frequency_penalty: Optional[float] = Field(0.0, description="Frequency penalty")
    presence_penalty: Optional[float] = Field(0.0, description="Presence penalty")

    @validator('model')
    def validate_model(cls, v):
        if v != "cursor-agent-local":
            raise ValueError('Model harus "cursor-agent-local"')
        return v

    @validator('max_tokens')
    def validate_max_tokens(cls, v):
        if v is not None and (v < 1 or v > 8192):
            raise ValueError('max_tokens harus antara 1 dan 8192')
        return v

    @validator('temperature')
    def validate_temperature(cls, v):
        if v is not None and (v < 0 or v > 2):
            raise ValueError('temperature harus antara 0 dan 2')
        return v

class ChatChoice(BaseModel):
    """Chat completion choice"""
    index: int
    message: Optional[ChatMessage] = None
    delta: Optional[Dict[str, Any]] = None
    finish_reason: Optional[str] = None

class ChatCompletionResponse(BaseModel):
    """Chat completion response"""
    id: str
    object: str = "chat.completion"
    created: int
    model: str
    choices: List[ChatChoice]
    usage: Optional[Dict[str, int]] = None

class ModelInfo(BaseModel):
    """Model information"""
    id: str
    object: str = "model"
    created: int
    owned_by: str
    permission: List[Dict[str, Any]] = []
    root: str
    parent: Optional[str] = None

class ModelListResponse(BaseModel):
    """Model list response"""
    object: str = "list"
    data: List[ModelInfo]

# ============================================================================
# Session Models
# ============================================================================

class Session(BaseModel):
    """Session information"""
    session_id: str
    client_ip: str
    model: str
    max_tokens: int
    created_at: datetime
    expires_at: datetime
    last_activity: datetime
    request_count: int = 0
    total_tokens: int = 0

# ============================================================================
# Health & Monitoring Models
# ============================================================================

class MCPConnectionStatus(BaseModel):
    """MCP connection status"""
    status: str
    last_ping: Optional[str] = None
    messages_sent: int = 0
    messages_received: int = 0
    pending_requests: int = 0
    active_streams: int = 0

class HealthResponse(BaseModel):
    """Health check response"""
    status: str
    timestamp: datetime
    version: str
    uptime_seconds: int
    active_sessions: int
    mcp_connection: MCPConnectionStatus

# ============================================================================
# Rate Limiting Models
# ============================================================================

class RateLimitInfo(BaseModel):
    """Rate limit information"""
    requests_per_minute: int
    tokens_per_minute: int
    current_requests: int
    current_tokens: int
    reset_time: datetime

# ============================================================================
# Error Models
# ============================================================================

class ErrorDetail(BaseModel):
    """Error detail"""
    message: str
    type: str
    param: Optional[str] = None
    code: Optional[str] = None

class ErrorResponse(BaseModel):
    """Error response"""
    error: ErrorDetail
```

---

## 🚦 Rate Limiter (`local_provider/rate_limiter.py`)

```python
"""
Rate Limiter - Pembatasan request dan token
"""

import asyncio
import time
from collections import defaultdict
from typing import Dict, List, Tuple
import logging

logger = logging.getLogger(__name__)

class RateLimiter:
    def __init__(self, requests_per_minute: int, tokens_per_minute: int):
        self.rpm_limit = requests_per_minute
        self.tpm_limit = tokens_per_minute

        # Request tracking
        self.request_history: Dict[str, List[float]] = defaultdict(list)
        self.token_history: Dict[str, List[Tuple[float, int]]] = defaultdict(list)

        # Statistics
        self.total_requests = 0
        self.successful_requests = 0
        self.failed_requests = 0
        self.rate_limited_requests = 0

    async def check_rate_limit(
        self,
        client_id: str,
        tokens: int = 1
    ) -> Tuple[bool, Dict[str, any]]:
        """Check rate limits untuk client"""

        now = time.time()
        window_start = now - 60  # 1 minute window

        # Clean old entries
        self.request_history[client_id] = [
            req_time for req_time in self.request_history[client_id]
            if req_time > window_start
        ]

        self.token_history[client_id] = [
            (req_time, token_count) for req_time, token_count in self.token_history[client_id]
            if req_time > window_start
        ]

        # Check request rate limit
        request_count = len(self.request_history[client_id])
        if request_count >= self.rpm_limit:
            self.rate_limited_requests += 1
            return False, {
                "error": "Request rate limit exceeded",
                "limit": self.rpm_limit,
                "current": request_count,
                "window": "1 minute",
                "retry_after": 60
            }

        # Check token rate limit
        total_tokens = sum(token_count for _, token_count in self.token_history[client_id])
        if total_tokens + tokens > self.tpm_limit:
            self.rate_limited_requests += 1
            return False, {
                "error": "Token rate limit exceeded",
                "limit": self.tpm_limit,
                "current": total_tokens,
                "requested": tokens,
                "window": "1 minute",
                "retry_after": 60
            }

        # Update counters
        self.total_requests += 1
        self.successful_requests += 1

        # Add current request
        self.request_history[client_id].append(now)
        self.token_history[client_id].append((now, tokens))

        return True, {
            "requests_remaining": self.rpm_limit - request_count - 1,
            "tokens_remaining": self.tpm_limit - total_tokens - tokens,
            "reset_time": window_start + 60
        }

    async def cleanup_task(self):
        """Background task untuk cleanup old data"""
        while True:
            try:
                await asyncio.sleep(300)  # Cleanup setiap 5 menit
                await self._cleanup_old_data()
            except Exception as e:
                logger.error(f"Rate limiter cleanup error: {e}")

    async def _cleanup_old_data(self):
        """Remove old rate limit data"""
        now = time.time()
        window_start = now - 3600  # Keep 1 hour of data

        # Cleanup request history
        for client_id in list(self.request_history.keys()):
            self.request_history[client_id] = [
                req_time for req_time in self.request_history[client_id]
                if req_time > window_start
            ]

            # Remove empty entries
            if not self.request_history[client_id]:
                del self.request_history[client_id]

        # Cleanup token history
        for client_id in list(self.token_history.keys()):
            self.token_history[client_id] = [
                (req_time, tokens) for req_time, tokens in self.token_history[client_id]
                if req_time > window_start
            ]

            # Remove empty entries
            if not self.token_history[client_id]:
                del self.token_history[client_id]

        logger.debug("Rate limiter data cleanup completed")
```

---

## 🔍 Audit Logger (`local_provider/audit.py`)

```python
"""
Audit Logger - Security dan activity logging
"""

import json
import logging
from datetime import datetime
from typing import Any, Dict

class AuditLogger:
    def __init__(self, logger_name: str = "audit"):
        self.logger = logging.getLogger(logger_name)

        # Setup file handler untuk audit logs
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
            "source": "local_provider",
            **data
        }
        self.logger.info(json.dumps(audit_entry))

    def log_authentication(self, client_ip: str, api_key_prefix: str, success: bool):
        """Log authentication attempt"""
        self._log_event("authentication", {
            "client_ip": client_ip,
            "api_key_prefix": api_key_prefix,
            "success": success,
            "user_agent": "VoidEditor"  # Could be extracted from request
        })

    def log_model_request(self, client_ip: str, model: str, tokens: int, success: bool):
        """Log model request"""
        self._log_event("model_request", {
            "client_ip": client_ip,
            "model": model,
            "estimated_tokens": tokens,
            "success": success
        })

    def log_rate_limit(self, client_ip: str, limit_type: str, current: int, limit: int):
        """Log rate limit violation"""
        self._log_event("rate_limit_exceeded", {
            "client_ip": client_ip,
            "limit_type": limit_type,
            "current_value": current,
            "limit_value": limit
        })

    def log_session_created(self, session_id: str, client_ip: str, model: str):
        """Log session creation"""
        self._log_event("session_created", {
            "session_id": session_id,
            "client_ip": client_ip,
            "model": model
        })

    def log_session_expired(self, session_id: str, client_ip: str):
        """Log session expiration"""
        self._log_event("session_expired", {
            "session_id": session_id,
            "client_ip": client_ip
        })

    def log_mcp_connection(self, status: str, details: str):
        """Log MCP connection events"""
        self._log_event("mcp_connection", {
            "status": status,
            "details": details
        })

    def log_security_violation(self, client_ip: str, violation_type: str, details: str):
        """Log security violation"""
        self._log_event("security_violation", {
            "client_ip": client_ip,
            "violation_type": violation_type,
            "details": details
        })
```

---

**Next**: Lihat [TESTING.md](TESTING.md) untuk strategi testing dan test cases yang komprehensif
