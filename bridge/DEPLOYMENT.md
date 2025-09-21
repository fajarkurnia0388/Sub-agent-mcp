# MCP VoidEditor Bridge — Deployment Guide

> **Language**: [🇺🇸 English](DEPLOYMENT.md) | [🇮🇩 Bahasa Indonesia](docs_id/DEPLOYMENT.md)

**Production deployment and security hardening for the MCP Bridge system**

## 🚀 Deployment Overview

The MCP Bridge is designed for secure localhost deployment, providing controlled access between Cursor agents and VoidEditor instances.

### Production Topology

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│   Cursor IDE    │◄──►│   MCP Bridge    │◄──►│   VoidEditor    │
│                 │    │  (FastAPI)      │    │   (Plugin)      │
│  Agent/MCP      │    │                 │    │                 │
│                 │    │ 127.0.0.1:8787  │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                        │                        │
        │                        │                        │
        v                        v                        v
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│   Log Files     │    │  Session Store  │    │  File System    │
│                 │    │   (Memory)      │    │  (Sandboxed)    │
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Security Boundaries

1. **Network**: Localhost-only binding (127.0.0.1)
2. **Authentication**: MCP token + session tokens
3. **Authorization**: Scope-based permissions
4. **File System**: Root path restrictions
5. **Process**: User-level process isolation

---

## 🛠 System Requirements

### Minimum Requirements

| Component   | Requirement                                        |
| ----------- | -------------------------------------------------- |
| **OS**      | Windows 10/11, macOS 10.15+, Linux (Ubuntu 20.04+) |
| **Python**  | 3.9+                                               |
| **RAM**     | 512MB available                                    |
| **Disk**    | 100MB free space                                   |
| **Network** | Localhost access only                              |

### Recommended Requirements

| Component  | Requirement            |
| ---------- | ---------------------- |
| **OS**     | Latest stable versions |
| **Python** | 3.11+                  |
| **RAM**    | 1GB available          |
| **Disk**   | 1GB free space         |

### Dependencies

```txt
fastapi>=0.104.0
uvicorn>=0.24.0
websockets>=12.0
pydantic>=2.5.0
python-jose[cryptography]>=3.3.0
cryptography>=41.0.0
pyyaml>=6.0
requests>=2.31.0
psutil>=5.9.0
```

---

## 📦 Installation Methods

### Method 1: Package Installation (Recommended)

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install package
pip install mcp-voideditor-bridge

# Generate configuration
mcp-bridge init

# Start the bridge
mcp-bridge start
```

### Method 2: From Source

```bash
# Clone repository
git clone https://github.com/your-org/mcp-voideditor-bridge.git
cd mcp-voideditor-bridge

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Copy configuration template
cp config/config.yaml.example config/config.yaml

# Start development server
python -m bridge.main
```

### Method 3: Docker Deployment

```bash
# Pull official image
docker pull mcpbridge/voideditor-bridge:latest

# Run container
docker run -d \
  --name mcp-bridge \
  -p 127.0.0.1:8787:8787 \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/logs:/app/logs \
  mcpbridge/voideditor-bridge:latest
```

---

## ⚙️ Configuration

### Environment Variables

```bash
# .env file
MCP_BRIDGE_PORT=8787
MCP_BRIDGE_HOST=127.0.0.1
MCP_BRIDGE_TOKEN=your-secure-token-here
MCP_BRIDGE_LOG_LEVEL=INFO
MCP_BRIDGE_SESSION_TTL=300
MCP_BRIDGE_MAX_SESSIONS=10
MCP_BRIDGE_RATE_LIMIT=10
MCP_BRIDGE_RATE_WINDOW=60
```

### Configuration File (`config/config.yaml`)

```yaml
# Bridge Configuration
bridge:
  host: "127.0.0.1"
  port: 8787
  debug: false

# Security Configuration
security:
  mcp_token: "${MCP_BRIDGE_TOKEN}"
  session_ttl: 300
  max_sessions: 10
  allowed_origins:
    - "http://127.0.0.1:*"
    - "ws://127.0.0.1:*"

# Rate Limiting
rate_limiting:
  enabled: true
  requests_per_window: 10
  window_seconds: 60

# Logging Configuration
logging:
  level: "INFO"
  format: "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  handlers:
    - console
    - file
  file_config:
    filename: "logs/bridge.log"
    max_bytes: 10485760 # 10MB
    backup_count: 5

# Session Management
sessions:
  cleanup_interval: 60
  max_idle_time: 600
  store_type: "memory" # Future: redis, file

# Plugin Management
plugins:
  registration_timeout: 30
  heartbeat_interval: 10
  max_plugins: 5

# File System Security
filesystem:
  max_file_size: 104857600 # 100MB
  max_edit_size: 102400 # 100KB
  allowed_extensions:
    - ".py"
    - ".js"
    - ".ts"
    - ".jsx"
    - ".tsx"
    - ".json"
    - ".yaml"
    - ".yml"
    - ".md"
    - ".txt"
    - ".css"
    - ".html"
    - ".xml"
    - ".sql"
  blocked_paths:
    - "**/.git/**"
    - "**/node_modules/**"
    - "**/__pycache__/**"
    - "**/venv/**"
    - "**/env/**"
    - "**/.env*"
```

### Secure Token Generation

```bash
# Generate cryptographically secure token
python -c "import secrets; print(secrets.token_urlsafe(64))"

# Or using openssl
openssl rand -base64 64 | tr -d '\n'

# Or using a dedicated script
python tools/generate_token.py
```

---

## 🔒 Security Hardening

### 1. Network Security

**Localhost Binding Only**

```python
# Ensure binding only to localhost
app.run(host="127.0.0.1", port=8787)
```

**Firewall Configuration (Linux)**

```bash
# Allow only localhost connections
sudo ufw deny 8787
sudo ufw allow from 127.0.0.1 to any port 8787
```

**Firewall Configuration (Windows)**

```powershell
# Create inbound rule for localhost only
New-NetFirewallRule -DisplayName "MCP Bridge" -Direction Inbound -Protocol TCP -LocalPort 8787 -LocalAddress 127.0.0.1 -Action Allow
```

### 2. Authentication & Authorization

**Token Security**

```yaml
security:
  # Use cryptographically secure tokens
  mcp_token: "{{ secrets.generate_token(64) }}"

  # Token rotation (future enhancement)
  token_rotation:
    enabled: false
    interval_hours: 24

  # Session token security
  session_tokens:
    algorithm: "HS256"
    expiry: 300
    secure: true
```

**Scope Validation**

```python
def validate_scope(scope: str, allowed_scopes: list) -> bool:
    """Validate that requested scope is allowed"""
    if scope not in VALID_SCOPES:
        return False
    return scope in allowed_scopes
```

### 3. Input Validation

**Request Validation**

```python
from pydantic import BaseModel, validator
from typing import List
import re

class AccessRequest(BaseModel):
    agent_id: str
    scopes: List[str]
    roots: List[str]
    reason: str

    @validator('agent_id')
    def validate_agent_id(cls, v):
        if not re.match(r'^[a-zA-Z0-9\-_]+$', v):
            raise ValueError('Invalid agent_id format')
        return v

    @validator('scopes')
    def validate_scopes(cls, v):
        invalid = [s for s in v if s not in VALID_SCOPES]
        if invalid:
            raise ValueError(f'Invalid scopes: {invalid}')
        return v
```

**Path Validation**

```python
import os
from pathlib import Path

def validate_path(path: str, allowed_roots: List[str]) -> bool:
    """Validate that path is within allowed roots"""
    abs_path = os.path.abspath(path)

    for root in allowed_roots:
        abs_root = os.path.abspath(root)
        try:
            Path(abs_path).relative_to(Path(abs_root))
            return True
        except ValueError:
            continue

    return False
```

### 4. Rate Limiting Implementation

```python
from collections import defaultdict
import time
import asyncio

class RateLimiter:
    def __init__(self, requests_per_window: int, window_seconds: int):
        self.requests_per_window = requests_per_window
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)

    async def is_allowed(self, identifier: str) -> bool:
        now = time.time()
        window_start = now - self.window_seconds

        # Clean old requests
        self.requests[identifier] = [
            req_time for req_time in self.requests[identifier]
            if req_time > window_start
        ]

        # Check if limit exceeded
        if len(self.requests[identifier]) >= self.requests_per_window:
            return False

        # Add current request
        self.requests[identifier].append(now)
        return True
```

### 5. Audit Logging

```python
import logging
import json
from datetime import datetime

class AuditLogger:
    def __init__(self, logger_name: str = "audit"):
        self.logger = logging.getLogger(logger_name)

    def log_access_request(self, agent_id: str, scopes: list, result: str):
        self.logger.info(json.dumps({
            "event": "access_request",
            "timestamp": datetime.utcnow().isoformat(),
            "agent_id": agent_id,
            "scopes": scopes,
            "result": result
        }))

    def log_command(self, session_id: str, command: str, result: str):
        self.logger.info(json.dumps({
            "event": "command_executed",
            "timestamp": datetime.utcnow().isoformat(),
            "session_id": session_id,
            "command": command,
            "result": result
        }))
```

---

## 🚀 Production Deployment

### Systemd Service (Linux)

```ini
# /etc/systemd/system/mcp-bridge.service
[Unit]
Description=MCP VoidEditor Bridge
After=network.target

[Service]
Type=exec
User=mcpbridge
Group=mcpbridge
WorkingDirectory=/opt/mcp-bridge
Environment=PATH=/opt/mcp-bridge/venv/bin
ExecStart=/opt/mcp-bridge/venv/bin/python -m bridge.main
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# Security
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/mcp-bridge/logs
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl enable mcp-bridge
sudo systemctl start mcp-bridge
sudo systemctl status mcp-bridge
```

### Windows Service

```powershell
# Install using NSSM (Non-Sucking Service Manager)
nssm install MCPBridge "C:\mcp-bridge\venv\Scripts\python.exe"
nssm set MCPBridge Arguments "-m bridge.main"
nssm set MCPBridge AppDirectory "C:\mcp-bridge"
nssm set MCPBridge DisplayName "MCP VoidEditor Bridge"
nssm set MCPBridge Description "Secure bridge between Cursor and VoidEditor"
nssm start MCPBridge
```

### macOS LaunchAgent

```xml
<!-- ~/Library/LaunchAgents/com.mcpbridge.agent.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.mcpbridge.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/python3</string>
        <string>-m</string>
        <string>bridge.main</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/usr/local/opt/mcp-bridge</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/usr/local/var/log/mcp-bridge.log</string>
    <key>StandardErrorPath</key>
    <string>/usr/local/var/log/mcp-bridge.error.log</string>
</dict>
</plist>
```

```bash
# Load and start service
launchctl load ~/Library/LaunchAgents/com.mcpbridge.agent.plist
launchctl start com.mcpbridge.agent
```

---

## 📊 Monitoring & Health Checks

### Health Check Endpoint

```python
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0",
        "active_sessions": len(session_manager.sessions),
        "uptime_seconds": time.time() - start_time
    }
```

### Monitoring Dashboard

```python
@app.get("/metrics")
async def metrics():
    return {
        "sessions": {
            "active": len(session_manager.sessions),
            "total_created": session_manager.total_created,
            "total_expired": session_manager.total_expired
        },
        "requests": {
            "total": request_counter.total,
            "successful": request_counter.successful,
            "failed": request_counter.failed
        },
        "performance": {
            "avg_response_time_ms": performance_monitor.avg_response_time,
            "memory_usage_mb": psutil.Process().memory_info().rss / 1024 / 1024
        }
    }
```

### Log Monitoring

```bash
# Monitor logs in real-time
tail -f logs/bridge.log

# Search for errors
grep "ERROR" logs/bridge.log

# Monitor specific session
grep "sess-123e4567" logs/bridge.log
```

---

## 🔄 Backup & Recovery

### Configuration Backup

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/backups/mcp-bridge"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Backup configuration
cp -r config/ "$BACKUP_DIR/config_$DATE/"

# Backup logs (last 7 days)
find logs/ -name "*.log" -mtime -7 -exec cp {} "$BACKUP_DIR/logs_$DATE/" \;

# Compress backup
tar -czf "$BACKUP_DIR/backup_$DATE.tar.gz" "$BACKUP_DIR/config_$DATE" "$BACKUP_DIR/logs_$DATE"

# Cleanup old backups (keep last 30 days)
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +30 -delete
```

### Disaster Recovery

1. **Service Recovery**

   ```bash
   # Stop service
   sudo systemctl stop mcp-bridge

   # Restore configuration
   tar -xzf backup_20240101_120000.tar.gz
   cp -r config_20240101_120000/* config/

   # Restart service
   sudo systemctl start mcp-bridge
   ```

2. **Data Recovery**
   - Sessions are stored in memory (ephemeral by design)
   - No persistent data to recover
   - Configuration and logs are the only persistent data

---

## 🚨 Incident Response

### Security Incident Response

1. **Immediate Response**

   ```bash
   # Stop the service immediately
   sudo systemctl stop mcp-bridge

   # Revoke all active sessions
   curl -X POST http://127.0.0.1:8787/mcp/revoke_all \
     -H "Authorization: Bearer $MCP_TOKEN"
   ```

2. **Investigation**

   ```bash
   # Check audit logs
   grep "SECURITY" logs/audit.log

   # Check active connections
   netstat -tulpn | grep 8787

   # Check process information
   ps aux | grep bridge
   ```

3. **Recovery**

   ```bash
   # Rotate tokens
   python tools/generate_token.py > new_token.txt

   # Update configuration
   export MCP_BRIDGE_TOKEN=$(cat new_token.txt)

   # Restart service
   sudo systemctl start mcp-bridge
   ```

### Performance Issues

1. **High Memory Usage**

   ```bash
   # Check memory usage
   ps aux | grep bridge

   # Check session count
   curl http://127.0.0.1:8787/metrics

   # Restart if necessary
   sudo systemctl restart mcp-bridge
   ```

2. **High CPU Usage**

   ```bash
   # Check CPU usage
   top -p $(pgrep -f bridge)

   # Check for stuck sessions
   curl http://127.0.0.1:8787/mcp/sessions
   ```

---

## 📋 Deployment Checklist

### Pre-Deployment

- [ ] Python 3.9+ installed
- [ ] Virtual environment created
- [ ] Dependencies installed
- [ ] Configuration file created
- [ ] Secure token generated
- [ ] Firewall rules configured
- [ ] Log directories created with proper permissions
- [ ] User account created (if running as service)

### Security Configuration

- [ ] MCP token is cryptographically secure (64+ characters)
- [ ] Binding is localhost-only (127.0.0.1)
- [ ] Rate limiting is enabled
- [ ] File system restrictions are in place
- [ ] Audit logging is configured
- [ ] Token is not committed to version control
- [ ] Proper file permissions set (600 for token files)

### Service Configuration

- [ ] Service definition created
- [ ] Auto-start on boot configured
- [ ] Restart policy configured
- [ ] Log rotation configured
- [ ] Health check endpoint accessible
- [ ] Metrics endpoint accessible

### Testing

- [ ] Health check returns "healthy"
- [ ] Access request flow works
- [ ] Session creation works
- [ ] WebSocket connections work
- [ ] Rate limiting triggers correctly
- [ ] Invalid tokens are rejected
- [ ] File path validation works
- [ ] Audit logs are generated

### Monitoring

- [ ] Log monitoring configured
- [ ] Metrics collection configured
- [ ] Alerting configured
- [ ] Backup scripts scheduled
- [ ] Incident response plan documented

---

**Next**: See [IMPLEMENTATION.md](IMPLEMENTATION.md) for code examples and implementation details
