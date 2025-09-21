# MCP Local Provider — Panduan Deployment

> **Bahasa**: [🇺🇸 English](../DEPLOYMENT.md) | [🇮🇩 Bahasa Indonesia](DEPLOYMENT.md)

**Panduan deployment produksi dan hardening keamanan untuk sistem MCP Local Provider**

## 🚀 Gambaran Deployment

MCP Local Provider dirancang untuk deployment localhost yang aman, menyediakan akses terkontrol antara agen Cursor dan instansi VoidEditor.

### Topologi Produksi

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│   VoidEditor    │◄──►│ Provider Adapter│◄──►│   Cursor IDE    │
│  (AI Client)    │    │   (FastAPI)     │    │  (MCP Agent)    │
│                 │    │                 │    │                 │
│                 │    │ 127.0.0.1:8788  │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                        │                        │
        │                        │                        │
        v                        v                        v
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│  Config Files   │    │  Session Store  │    │   Log Files     │
│                 │    │   (Memory)      │    │   (Audit)       │
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Boundary Keamanan

1. **Network**: Binding hanya localhost (127.0.0.1)
2. **Authentication**: API key + session tokens
3. **Authorization**: Rate limiting berbasis client
4. **Process**: Isolasi process level user
5. **Data**: Session efemeral tanpa persistent storage

---

## 🛠 Persyaratan Sistem

### Persyaratan Minimum

| Komponen    | Requirement                                        |
| ----------- | -------------------------------------------------- |
| **OS**      | Windows 10/11, macOS 10.15+, Linux (Ubuntu 20.04+) |
| **Python**  | 3.9+                                               |
| **RAM**     | 256MB tersedia                                     |
| **Disk**    | 50MB ruang kosong                                  |
| **Network** | Akses localhost saja                               |

### Persyaratan Direkomendasikan

| Komponen   | Requirement          |
| ---------- | -------------------- |
| **OS**     | Versi stabil terbaru |
| **Python** | 3.11+                |
| **RAM**    | 512MB tersedia       |
| **Disk**   | 200MB ruang kosong   |

### Dependencies

```txt
fastapi>=0.104.0
uvicorn>=0.24.0
websockets>=12.0
pydantic>=2.5.0
httpx>=0.25.0
cryptography>=41.0.0
pyyaml>=6.0
requests>=2.31.0
psutil>=5.9.0
```

---

## 📦 Metode Instalasi

### Metode 1: Instalasi Package (Direkomendasikan)

```bash
# Buat virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install package
pip install mcp-local-provider

# Generate konfigurasi
mcp-provider init

# Start provider
mcp-provider start
```

### Metode 2: Dari Source

```bash
# Clone repository
git clone https://github.com/your-org/mcp-local-provider.git
cd mcp-local-provider

# Buat virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Salin template konfigurasi
cp config/config.yaml.example config/config.yaml

# Start development server
python -m local_provider.adapter
```

### Metode 3: Docker Deployment

```bash
# Pull official image
docker pull mcpprovider/local-provider:latest

# Run container
docker run -d \
  --name mcp-local-provider \
  -p 127.0.0.1:8788:8788 \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/logs:/app/logs \
  mcpprovider/local-provider:latest
```

---

## ⚙️ Konfigurasi

### Environment Variables

```bash
# .env file
LOCAL_PROVIDER_PORT=8788
LOCAL_PROVIDER_HOST=127.0.0.1
LOCAL_PROVIDER_API_KEY=your-secure-api-key-here
LOCAL_PROVIDER_LOG_LEVEL=INFO
LOCAL_PROVIDER_SESSION_TTL=300
LOCAL_PROVIDER_MAX_SESSIONS=10
LOCAL_PROVIDER_RATE_LIMIT_RPM=60
LOCAL_PROVIDER_RATE_LIMIT_TPM=100000
```

### File Konfigurasi (`config/config.yaml`)

```yaml
# Provider Configuration
provider:
  host: "127.0.0.1"
  port: 8788
  model_name: "cursor-agent-local"
  debug: false

# Security Configuration
security:
  api_key: "${LOCAL_PROVIDER_API_KEY}"
  session_ttl: 300
  max_sessions: 10
  allowed_origins:
    - "http://127.0.0.1:*"
    - "ws://127.0.0.1:*"

# Rate Limiting
rate_limiting:
  enabled: true
  requests_per_minute: 60
  tokens_per_minute: 100000
  burst_allowance: 10

# Logging Configuration
logging:
  level: "INFO"
  format: "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  handlers:
    - console
    - file
  file_config:
    filename: "logs/provider.log"
    max_bytes: 10485760 # 10MB
    backup_count: 5

# Session Management
sessions:
  cleanup_interval: 60
  max_idle_time: 300
  store_type: "memory" # Future: redis, file

# MCP Configuration
mcp:
  bridge_url: "ws://127.0.0.1:8787"
  agent_id: "local-provider-1"
  connection_timeout: 30
  heartbeat_interval: 30
  reconnect_attempts: 5
  reconnect_delay: 5

# Model Configuration
model:
  max_tokens: 2048
  temperature: 0.7
  top_p: 1.0
  frequency_penalty: 0.0
  presence_penalty: 0.0
  stream_enabled: true

# Performance Tuning
performance:
  worker_threads: 4
  max_concurrent_requests: 50
  request_timeout: 30
  response_timeout: 60
  memory_limit_mb: 512
```

### Generate API Key Aman

```bash
# Generate cryptographically secure API key
python -c "import secrets; print(secrets.token_urlsafe(64))"

# Atau menggunakan openssl
openssl rand -base64 64 | tr -d '\n'

# Atau menggunakan script khusus
python tools/generate_api_key.py
```

---

## 🔒 Hardening Keamanan

### 1. Keamanan Network

**Binding Localhost Saja**

```python
# Pastikan binding hanya ke localhost
app.run(host="127.0.0.1", port=8788)
```

**Konfigurasi Firewall (Linux)**

```bash
# Hanya izinkan koneksi localhost
sudo ufw deny 8788
sudo ufw allow from 127.0.0.1 to any port 8788
```

**Konfigurasi Firewall (Windows)**

```powershell
# Buat inbound rule untuk localhost saja
New-NetFirewallRule -DisplayName "MCP Local Provider" -Direction Inbound -Protocol TCP -LocalPort 8788 -LocalAddress 127.0.0.1 -Action Allow
```

### 2. Autentikasi & Otorisasi

**Keamanan API Key**

```yaml
security:
  # Gunakan API key yang cryptographically secure
  api_key: "{{ secrets.generate_api_key(64) }}"

  # Key rotation (enhancement masa depan)
  key_rotation:
    enabled: false
    interval_hours: 24

  # Session token security
  session_tokens:
    algorithm: "HS256"
    expiry: 300
    secure: true
```

**Validasi API Key**

```python
import hashlib
import secrets

def validate_api_key(provided_key: str, stored_hash: str) -> bool:
    """Validate API key dengan constant-time comparison"""
    provided_hash = hashlib.sha256(provided_key.encode()).hexdigest()
    return secrets.compare_digest(provided_hash, stored_hash)
```

### 3. Validasi Input

**Validasi Request**

```python
from pydantic import BaseModel, validator
from typing import List, Optional
import re

class ChatCompletionRequest(BaseModel):
    model: str
    messages: List[dict]
    max_tokens: Optional[int] = 2048
    temperature: Optional[float] = 0.7

    @validator('model')
    def validate_model(cls, v):
        if v != "cursor-agent-local":
            raise ValueError('Invalid model name')
        return v

    @validator('max_tokens')
    def validate_max_tokens(cls, v):
        if v is not None and (v < 1 or v > 8192):
            raise ValueError('max_tokens must be between 1 and 8192')
        return v

    @validator('temperature')
    def validate_temperature(cls, v):
        if v is not None and (v < 0 or v > 2):
            raise ValueError('temperature must be between 0 and 2')
        return v
```

**Sanitasi Content**

```python
import html
import re

def sanitize_content(content: str) -> str:
    """Sanitize user input untuk mencegah injection"""
    # HTML escape
    content = html.escape(content)

    # Remove potentially dangerous patterns
    content = re.sub(r'<script[^>]*>.*?</script>', '', content, flags=re.IGNORECASE | re.DOTALL)
    content = re.sub(r'javascript:', '', content, flags=re.IGNORECASE)

    # Limit length
    if len(content) > 100000:  # 100KB limit
        content = content[:100000]

    return content
```

### 4. Rate Limiting Implementation

```python
import asyncio
import time
from collections import defaultdict
from typing import Dict, Tuple

class AdvancedRateLimiter:
    def __init__(self, requests_per_minute: int, tokens_per_minute: int):
        self.rpm_limit = requests_per_minute
        self.tpm_limit = tokens_per_minute
        self.request_history: Dict[str, List[float]] = defaultdict(list)
        self.token_history: Dict[str, List[Tuple[float, int]]] = defaultdict(list)

    async def check_rate_limit(self, client_id: str, tokens: int = 1) -> Tuple[bool, dict]:
        """Check both request and token rate limits"""
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
            return False, {
                "error": "Request rate limit exceeded",
                "limit": self.rpm_limit,
                "current": request_count,
                "window": "1 minute"
            }

        # Check token rate limit
        total_tokens = sum(token_count for _, token_count in self.token_history[client_id])
        if total_tokens + tokens > self.tpm_limit:
            return False, {
                "error": "Token rate limit exceeded",
                "limit": self.tpm_limit,
                "current": total_tokens,
                "requested": tokens,
                "window": "1 minute"
            }

        # Add current request
        self.request_history[client_id].append(now)
        self.token_history[client_id].append((now, tokens))

        return True, {
            "requests_remaining": self.rpm_limit - request_count - 1,
            "tokens_remaining": self.tpm_limit - total_tokens - tokens
        }
```

### 5. Audit Logging

```python
import logging
import json
from datetime import datetime
from typing import Any, Dict, Optional

class SecurityAuditLogger:
    def __init__(self, logger_name: str = "security_audit"):
        self.logger = logging.getLogger(logger_name)

        # Setup file handler untuk audit logs
        handler = logging.FileHandler("logs/security_audit.log")
        formatter = logging.Formatter(
            '%(asctime)s - SECURITY - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

    def _log_event(self, event_type: str, data: Dict[str, Any]):
        """Log security event"""
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
            "success": success
        })

    def log_rate_limit(self, client_ip: str, limit_type: str, current: int, limit: int):
        """Log rate limit violation"""
        self._log_event("rate_limit_exceeded", {
            "client_ip": client_ip,
            "limit_type": limit_type,
            "current": current,
            "limit": limit
        })

    def log_model_request(self, client_ip: str, model: str, tokens: int, success: bool):
        """Log model request"""
        self._log_event("model_request", {
            "client_ip": client_ip,
            "model": model,
            "tokens": tokens,
            "success": success
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

## 🚀 Deployment Produksi

### Systemd Service (Linux)

```ini
# /etc/systemd/system/mcp-local-provider.service
[Unit]
Description=MCP Local Provider
After=network.target

[Service]
Type=exec
User=mcpprovider
Group=mcpprovider
WorkingDirectory=/opt/mcp-local-provider
Environment=PATH=/opt/mcp-local-provider/venv/bin
ExecStart=/opt/mcp-local-provider/venv/bin/python -m local_provider.adapter
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# Security
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/mcp-local-provider/logs
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
# Enable dan start service
sudo systemctl enable mcp-local-provider
sudo systemctl start mcp-local-provider
sudo systemctl status mcp-local-provider
```

### Windows Service

```powershell
# Install menggunakan NSSM (Non-Sucking Service Manager)
nssm install MCPLocalProvider "C:\mcp-local-provider\venv\Scripts\python.exe"
nssm set MCPLocalProvider Arguments "-m local_provider.adapter"
nssm set MCPLocalProvider AppDirectory "C:\mcp-local-provider"
nssm set MCPLocalProvider DisplayName "MCP Local Provider"
nssm set MCPLocalProvider Description "Local AI model provider for VoidEditor"
nssm start MCPLocalProvider
```

### macOS LaunchAgent

```xml
<!-- ~/Library/LaunchAgents/com.mcplocalprovider.agent.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.mcplocalprovider.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/python3</string>
        <string>-m</string>
        <string>local_provider.adapter</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/usr/local/opt/mcp-local-provider</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/usr/local/var/log/mcp-local-provider.log</string>
    <key>StandardErrorPath</key>
    <string>/usr/local/var/log/mcp-local-provider.error.log</string>
</dict>
</plist>
```

```bash
# Load dan start service
launchctl load ~/Library/LaunchAgents/com.mcplocalprovider.agent.plist
launchctl start com.mcplocalprovider.agent
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
        "uptime_seconds": time.time() - start_time,
        "active_sessions": len(session_manager.sessions),
        "mcp_connection": {
            "status": "connected" if mcp_client.is_connected() else "disconnected",
            "last_ping": mcp_client.last_ping_time.isoformat() if mcp_client.last_ping_time else None
        }
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
            "failed": request_counter.failed,
            "rate_limited": request_counter.rate_limited
        },
        "performance": {
            "avg_response_time_ms": performance_monitor.avg_response_time,
            "p95_response_time_ms": performance_monitor.p95_response_time,
            "p99_response_time_ms": performance_monitor.p99_response_time,
            "memory_usage_mb": psutil.Process().memory_info().rss / 1024 / 1024,
            "cpu_usage_percent": psutil.Process().cpu_percent()
        },
        "tokens": {
            "total_processed": token_counter.total,
            "avg_tokens_per_request": token_counter.average,
            "tokens_per_second": token_counter.rate
        }
    }
```

### Log Monitoring

```bash
# Monitor logs secara real-time
tail -f logs/provider.log

# Cari error
grep "ERROR" logs/provider.log

# Monitor specific session
grep "sess-123e4567" logs/provider.log

# Monitor security events
tail -f logs/security_audit.log | grep "authentication"
```

---

## 🔄 Backup & Recovery

### Configuration Backup

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/backups/mcp-local-provider"

# Buat backup directory
mkdir -p "$BACKUP_DIR"

# Backup configuration
cp -r config/ "$BACKUP_DIR/config_$DATE/"

# Backup logs (7 hari terakhir)
find logs/ -name "*.log" -mtime -7 -exec cp {} "$BACKUP_DIR/logs_$DATE/" \;

# Compress backup
tar -czf "$BACKUP_DIR/backup_$DATE.tar.gz" "$BACKUP_DIR/config_$DATE" "$BACKUP_DIR/logs_$DATE"

# Cleanup backup lama (simpan 30 hari)
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +30 -delete
```

### Disaster Recovery

1. **Service Recovery**

   ```bash
   # Stop service
   sudo systemctl stop mcp-local-provider

   # Restore configuration
   tar -xzf backup_20240101_120000.tar.gz
   cp -r config_20240101_120000/* config/

   # Restart service
   sudo systemctl start mcp-local-provider
   ```

2. **Data Recovery**
   - Session data disimpan dalam memory (ephemeral by design)
   - Tidak ada persistent data untuk di-recover
   - Configuration dan logs adalah satu-satunya persistent data

---

## 🚨 Incident Response

### Security Incident Response

1. **Immediate Response**

   ```bash
   # Stop service segera
   sudo systemctl stop mcp-local-provider

   # Revoke semua active sessions (jika ada persistent storage)
   curl -X POST http://127.0.0.1:8788/admin/revoke_all_sessions \
     -H "Authorization: Bearer $ADMIN_TOKEN"
   ```

2. **Investigation**

   ```bash
   # Check audit logs
   grep "SECURITY" logs/security_audit.log

   # Check active connections
   netstat -tulpn | grep 8788

   # Check process information
   ps aux | grep local_provider
   ```

3. **Recovery**

   ```bash
   # Rotate API keys
   python tools/generate_api_key.py > new_api_key.txt

   # Update configuration
   export LOCAL_PROVIDER_API_KEY=$(cat new_api_key.txt)

   # Restart service
   sudo systemctl start mcp-local-provider
   ```

### Performance Issues

1. **High Memory Usage**

   ```bash
   # Check memory usage
   ps aux | grep local_provider

   # Check session count
   curl http://127.0.0.1:8788/metrics

   # Restart jika perlu
   sudo systemctl restart mcp-local-provider
   ```

2. **High CPU Usage**

   ```bash
   # Check CPU usage
   top -p $(pgrep -f local_provider)

   # Check untuk stuck requests
   curl http://127.0.0.1:8788/health
   ```

---

## 📋 Deployment Checklist

### Pre-Deployment

- [ ] Python 3.9+ installed
- [ ] Virtual environment created
- [ ] Dependencies installed
- [ ] Configuration file created
- [ ] Secure API key generated
- [ ] Firewall rules configured
- [ ] Log directories created dengan proper permissions
- [ ] User account created (jika running sebagai service)

### Security Configuration

- [ ] API key cryptographically secure (64+ karakter)
- [ ] Binding hanya localhost (127.0.0.1)
- [ ] Rate limiting enabled
- [ ] Input validation di-implement
- [ ] Audit logging configured
- [ ] API key tidak di-commit ke version control
- [ ] Proper file permissions set (600 untuk key files)

### Service Configuration

- [ ] Service definition created
- [ ] Auto-start on boot configured
- [ ] Restart policy configured
- [ ] Log rotation configured
- [ ] Health check endpoint accessible
- [ ] Metrics endpoint accessible

### Testing

- [ ] Health check mengembalikan "healthy"
- [ ] Chat completion flow bekerja
- [ ] Model list endpoint bekerja
- [ ] Streaming response bekerja
- [ ] Rate limiting trigger dengan benar
- [ ] Invalid API keys ditolak
- [ ] Input validation bekerja
- [ ] Audit logs ter-generate

### Monitoring

- [ ] Log monitoring configured
- [ ] Metrics collection configured
- [ ] Alerting configured
- [ ] Backup scripts scheduled
- [ ] Incident response plan documented

---

**Next**: Lihat [IMPLEMENTATION.md](IMPLEMENTATION.md) untuk contoh kode dan detail implementasi
