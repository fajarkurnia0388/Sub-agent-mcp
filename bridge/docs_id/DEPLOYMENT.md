# MCP VoidEditor Bridge — Panduan Deployment

> **Bahasa**: [🇺🇸 English](../DEPLOYMENT.md) | [🇮🇩 Bahasa Indonesia](DEPLOYMENT.md)

**Deployment produksi dan hardening keamanan untuk sistem MCP Bridge**

## 🚀 Gambaran Deployment

MCP Bridge dirancang untuk deployment localhost yang aman, memberikan akses terkontrol antara agen Cursor dan instance VoidEditor.

### Topologi Produksi

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

### Batas Keamanan

1. **Network**: Binding localhost-only (127.0.0.1)
2. **Authentication**: MCP token + session tokens
3. **Authorization**: Izin berbasis scope
4. **File System**: Pembatasan root path
5. **Process**: Isolasi proses tingkat user

---

## 🛠 Persyaratan Sistem

### Persyaratan Minimum

| Komponen    | Persyaratan                                        |
| ----------- | -------------------------------------------------- |
| **OS**      | Windows 10/11, macOS 10.15+, Linux (Ubuntu 20.04+) |
| **Python**  | 3.9+                                               |
| **RAM**     | 512MB tersedia                                     |
| **Storage** | 100MB untuk aplikasi + log                         |
| **Network** | Localhost connectivity                             |

### Persyaratan Rekomendasi

| Komponen    | Persyaratan                                  |
| ----------- | -------------------------------------------- |
| **OS**      | Windows 11, macOS 12+, Linux (Ubuntu 22.04+) |
| **Python**  | 3.11+                                        |
| **RAM**     | 1GB tersedia                                 |
| **Storage** | 500MB SSD untuk aplikasi + log               |
| **Network** | Localhost dengan firewall disabled           |

---

## 🔧 Environment Setup

### 1. Python Environment

```bash
# Buat virtual environment
python -m venv venv

# Aktifkan environment
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Environment Variables

Buat file `.env`:

```bash
# MCP Bridge Configuration
MCP_BRIDGE_HOST=127.0.0.1
MCP_BRIDGE_PORT=8787
MCP_BRIDGE_TOKEN=your-secure-token-here

# Security Settings
ALLOWED_ROOTS=/workspace,/projects
MAX_SESSIONS=10
SESSION_TTL=300

# Logging
LOG_LEVEL=INFO
LOG_FILE=bridge.log
AUDIT_LOG_FILE=audit.log

# Development (set to false in production)
DEBUG=false
```

### 3. Security Configuration

Buat file `security.yaml`:

```yaml
security:
  # Token configuration
  token_length: 32
  token_charset: "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

  # Session configuration
  max_sessions_per_agent: 3
  session_cleanup_interval: 60

  # Rate limiting
  rate_limit:
    requests_per_minute: 100
    burst_allowance: 20

  # File system restrictions
  filesystem:
    max_file_size: 10485760 # 10MB
    allowed_extensions: [".js", ".ts", ".py", ".md", ".json", ".yaml", ".yml"]
    blocked_paths: ["/etc", "/usr", "/bin", "/sbin"]
```

---

## 🚀 Production Deployment

### 1. System Service (Linux/macOS)

Buat file service `/etc/systemd/system/mcp-bridge.service`:

```ini
[Unit]
Description=MCP VoidEditor Bridge
After=network.target

[Service]
Type=simple
User=mcp-bridge
Group=mcp-bridge
WorkingDirectory=/opt/mcp-bridge
Environment=PATH=/opt/mcp-bridge/venv/bin
ExecStart=/opt/mcp-bridge/venv/bin/python -m bridge.mcp_bridge
Restart=always
RestartSec=5

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/mcp-bridge/logs

[Install]
WantedBy=multi-user.target
```

### 2. Windows Service

Buat file `install_service.py`:

```python
import win32serviceutil
import win32service
import win32event
import servicemanager
import sys
import os

class MCPBridgeService(win32serviceutil.ServiceFramework):
    _svc_name_ = "MCPBridge"
    _svc_display_name_ = "MCP VoidEditor Bridge"
    _svc_description_ = "Bridge service for Cursor-VoidEditor communication"

    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.hWaitStop = win32event.CreateEvent(None, 0, 0, None)

    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        win32event.SetEvent(self.hWaitStop)

    def SvcDoRun(self):
        servicemanager.LogMsg(servicemanager.EVENTLOG_INFORMATION_TYPE,
                              servicemanager.PYS_SERVICE_STARTED,
                              (self._svc_name_, ''))
        # Start MCP Bridge
        os.system("python -m bridge.mcp_bridge")

if __name__ == '__main__':
    win32serviceutil.HandleCommandLine(MCPBridgeService)
```

### 3. Docker Deployment

Buat file `Dockerfile`:

```dockerfile
FROM python:3.11-slim

# Security: Create non-root user
RUN groupadd -r mcp-bridge && useradd -r -g mcp-bridge mcp-bridge

# Install dependencies
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Security: Change ownership
RUN chown -R mcp-bridge:mcp-bridge /app

# Switch to non-root user
USER mcp-bridge

# Expose port
EXPOSE 8787

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://127.0.0.1:8787/health || exit 1

# Start application
CMD ["python", "-m", "bridge.mcp_bridge"]
```

Buat file `docker-compose.yml`:

```yaml
version: "3.8"

services:
  mcp-bridge:
    build: .
    ports:
      - "8787:8787"
    environment:
      - MCP_BRIDGE_HOST=0.0.0.0
      - MCP_BRIDGE_PORT=8787
      - MCP_BRIDGE_TOKEN=${MCP_BRIDGE_TOKEN}
    volumes:
      - ./logs:/app/logs
      - ./workspace:/workspace:ro
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

---

## 🔒 Security Hardening

### 1. Network Security

```bash
# Bind hanya ke localhost
MCP_BRIDGE_HOST=127.0.0.1

# Disable external access
iptables -A INPUT -p tcp --dport 8787 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 8787 -j DROP
```

### 2. Process Security

```bash
# Run sebagai user terbatas
sudo useradd -r -s /bin/false mcp-bridge
sudo chown -R mcp-bridge:mcp-bridge /opt/mcp-bridge

# Set file permissions
chmod 600 /opt/mcp-bridge/.env
chmod 644 /opt/mcp-bridge/security.yaml
```

### 3. Token Security

```python
# Generate secure token
import secrets
import string

def generate_token(length=32):
    alphabet = string.ascii_letters + string.digits
    return ''.join(secrets.choice(alphabet) for _ in range(length))

# Store token securely
token = generate_token()
# Store in secure key management system
```

### 4. File System Security

```python
# Validate file paths
import os
from pathlib import Path

def validate_path(path, allowed_roots):
    abs_path = os.path.abspath(path)
    for root in allowed_roots:
        if abs_path.startswith(os.path.abspath(root)):
            return True
    return False

# Example usage
allowed_roots = ["/workspace", "/projects"]
if not validate_path(requested_path, allowed_roots):
    raise SecurityError("Path not allowed")
```

---

## 📊 Monitoring & Logging

### 1. Application Logging

```python
import logging
import logging.handlers

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.handlers.RotatingFileHandler(
            'bridge.log', maxBytes=10485760, backupCount=5
        ),
        logging.StreamHandler()
    ]
)

# Audit logging
audit_logger = logging.getLogger('audit')
audit_handler = logging.handlers.RotatingFileHandler(
    'audit.log', maxBytes=10485760, backupCount=10
)
audit_logger.addHandler(audit_handler)
audit_logger.setLevel(logging.INFO)
```

### 2. Health Monitoring

```python
from fastapi import FastAPI
import psutil

app = FastAPI()

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "uptime": get_uptime(),
        "memory_usage": psutil.virtual_memory().percent,
        "cpu_usage": psutil.cpu_percent(),
        "active_sessions": get_active_sessions()
    }
```

### 3. Metrics Collection

```python
from prometheus_client import Counter, Histogram, Gauge

# Metrics
request_count = Counter('mcp_requests_total', 'Total requests', ['method', 'endpoint'])
request_duration = Histogram('mcp_request_duration_seconds', 'Request duration')
active_sessions = Gauge('mcp_active_sessions', 'Active sessions')
```

---

## 🚨 Incident Response

### 1. Security Incident Response

```bash
# Emergency stop
sudo systemctl stop mcp-bridge

# Check logs
tail -f /opt/mcp-bridge/logs/audit.log

# Revoke all sessions
curl -X POST http://127.0.0.1:8787/mcp/revoke_all \
  -H "Authorization: Bearer $MCP_TOKEN"

# Restart service
sudo systemctl start mcp-bridge
```

### 2. Performance Issues

```bash
# Check resource usage
top -p $(pgrep -f mcp_bridge)

# Check network connections
netstat -tulpn | grep 8787

# Check logs for errors
grep ERROR /opt/mcp-bridge/logs/bridge.log
```

### 3. Recovery Procedures

```bash
# Backup configuration
cp /opt/mcp-bridge/.env /opt/mcp-bridge/.env.backup
cp /opt/mcp-bridge/security.yaml /opt/mcp-bridge/security.yaml.backup

# Restore from backup
cp /opt/mcp-bridge/.env.backup /opt/mcp-bridge/.env
cp /opt/mcp-bridge/security.yaml.backup /opt/mcp-bridge/security.yaml

# Restart service
sudo systemctl restart mcp-bridge
```

---

## 📋 Deployment Checklist

### Pre-deployment

- [ ] Environment variables configured
- [ ] Security configuration reviewed
- [ ] Token generated and secured
- [ ] File permissions set correctly
- [ ] Network security configured
- [ ] Monitoring setup completed

### Post-deployment

- [ ] Service started successfully
- [ ] Health check passing
- [ ] Logs being written correctly
- [ ] Network connectivity verified
- [ ] Security policies enforced
- [ ] Monitoring alerts configured

### Maintenance

- [ ] Regular log rotation
- [ ] Security updates applied
- [ ] Performance monitoring
- [ ] Backup procedures tested
- [ ] Incident response plan updated

---

**Deployment ini dirancang untuk memberikan keamanan maksimal sambil menjaga kemudahan penggunaan untuk operasi sub-agent di VoidEditor.**
