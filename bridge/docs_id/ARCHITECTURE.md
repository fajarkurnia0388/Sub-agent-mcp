# MCP VoidEditor Bridge — Arsitektur & Desain

> **Bahasa**: [🇺🇸 English](../ARCHITECTURE.md) | [🇮🇩 Bahasa Indonesia](ARCHITECTURE.md)

**Arsitektur sistem detail, model data, dan interaksi komponen untuk MCP Bridge yang memungkinkan sub-agent di VoidEditor**

## 🏗️ Arsitektur Sistem

### Diagram Komponen Tingkat Tinggi

```mermaid
graph TB
    subgraph "Cursor IDE (Main Agent Environment)"
        CU[Cursor Agent/UI]
        MCP[MCP Client]
    end

    subgraph "MCP Bridge System"
        BA[Bridge API<br/>FastAPI Server]
        BM[Bridge Manager<br/>Session Controller]
        BW[Bridge WebSocket<br/>Agent/Plugin Proxy]
    end

    subgraph "VoidEditor IDE (Extended Environment)"
        VE[VoidEditor]
        VP[VoidEditor Plugin<br/>Sub-Agent Runtime]
    end

    CU -->|HTTP/SSE| BA
    MCP -->|WebSocket| BW
    VP -->|WebSocket| BW
    BA -.->|Control| BM
    BM -.->|Sessions| BW

    VP <-->|Full IDE Ops| VE
```

### Arsitektur Alur Data

```mermaid
sequenceDiagram
    participant C as Cursor Agent
    participant B as Bridge
    participant P as Plugin
    participant V as VoidEditor
    participant U as User

    Note over C,U: 1. Request Akses Sub-Agent
    C->>B: POST /mcp/request_access
    B->>U: SSE notification
    U->>B: POST /mcp/approve
    B->>C: SSE session_created

    Note over C,U: 2. Operasi Sub-Agent
    C->>B: WebSocket cmd message
    B->>P: Forward command
    P->>V: Execute IDE operation
    V->>P: Operation result
    P->>B: WebSocket result
    B->>C: Forward result

    Note over C,U: 3. Real-time Events
    V->>P: Editor state change
    P->>B: WebSocket event
    B->>C: Forward event
```

## 🧩 Komponen Utama

### 1. Cursor Agent (Main Agent Environment)

**Tanggung Jawab:**

- Mengirim request akses ke VoidEditor
- Menampilkan UI persetujuan pengguna
- Mengelola sesi sub-agent
- Menerima notifikasi real-time

**Teknologi:**

- MCP Client untuk komunikasi dengan Bridge
- UI untuk persetujuan dan monitoring
- WebSocket client untuk komunikasi real-time

### 2. MCP Bridge System

#### Bridge API (FastAPI Server)

**Tanggung Jawab:**

- Endpoint HTTP untuk request akses
- Manajemen sesi dan autentikasi
- Notifikasi SSE ke Cursor UI
- Logging dan audit

**Endpoint Utama:**

```python
POST /mcp/request_access    # Request akses sub-agent
POST /mcp/approve          # Persetujuan pengguna
POST /mcp/deny            # Penolakan pengguna
GET  /mcp/sessions        # Status sesi aktif
POST /mcp/revoke          # Pencabutan sesi
GET  /mcp/health          # Health check
```

#### Bridge Manager (Session Controller)

**Tanggung Jawab:**

- Manajemen lifecycle sesi
- Validasi scope dan izin
- Enforce policy keamanan
- Cleanup sesi expired

**Fitur:**

- Token-based authentication
- Scope-based access control
- TTL management
- Audit logging

#### Bridge WebSocket (Agent/Plugin Proxy)

**Tanggung Jawab:**

- Proxy komunikasi real-time
- Message routing antara agent dan plugin
- Session validation
- Error handling

### 3. VoidEditor Plugin (Sub-Agent Runtime)

**Tanggung Jawab:**

- Koneksi ke Bridge via WebSocket
- Eksekusi operasi IDE lengkap
- Real-time state synchronization
- Error reporting

**Kemampuan IDE:**

- File operations (CRUD)
- Code analysis dan formatting
- Terminal integration
- Version control (Git)
- Search dan replace
- Project management
- Real-time editor state sync

## 📊 Model Data

### Session Management

```python
class Session:
    session_id: str
    agent_id: str
    plugin_id: str
    scopes: List[str]
    root_path: str
    created_at: datetime
    expires_at: datetime
    status: SessionStatus
    last_activity: datetime
```

### MCP Message Format

```python
class MCPMessage:
    type: str  # "cmd" | "result" | "event"
    session_id: str
    timestamp: datetime
    data: Dict[str, Any]
```

### Command Structure

```python
class Command:
    action: str
    params: Dict[str, Any]
    scope: str
    timeout: Optional[int]
```

### Event Structure

```python
class Event:
    event_type: str
    data: Dict[str, Any]
    timestamp: datetime
```

## 🔄 Interaksi Komponen

### 1. Request Access Flow

```mermaid
sequenceDiagram
    participant A as Agent
    participant B as Bridge API
    participant M as Bridge Manager
    participant U as User UI
    participant W as Bridge WebSocket

    A->>B: POST /mcp/request_access
    B->>M: validate_request()
    M->>B: session_pending
    B->>U: SSE notification
    U->>B: POST /mcp/approve
    B->>M: create_session()
    M->>W: register_session()
    W->>A: WebSocket ready
    B->>U: SSE session_created
```

### 2. Command Execution Flow

```mermaid
sequenceDiagram
    participant A as Agent
    participant W as Bridge WebSocket
    participant P as Plugin
    participant V as VoidEditor

    A->>W: cmd message
    W->>W: validate_session()
    W->>P: forward_command()
    P->>V: execute_operation()
    V->>P: operation_result
    P->>W: result message
    W->>A: forward_result()
```

### 3. Real-time Event Flow

```mermaid
sequenceDiagram
    participant V as VoidEditor
    participant P as Plugin
    participant W as Bridge WebSocket
    participant A as Agent

    V->>P: editor_state_change
    P->>W: event message
    W->>W: validate_session()
    W->>A: forward_event()
```

## 🛡️ Keamanan & Isolasi

### Session Isolation

- Setiap sesi memiliki token unik
- Scope-based access control
- Root path restriction
- TTL enforcement

### Communication Security

- WebSocket over localhost only
- Token validation pada setiap request
- Message encryption (optional)
- Audit logging

### Error Handling

- Graceful degradation
- Session cleanup on errors
- User notification
- Recovery mechanisms

## 📈 Performance Considerations

### Scalability

- Stateless bridge design
- Session pooling
- Connection multiplexing
- Resource monitoring

### Optimization

- Message batching
- Compression
- Caching
- Lazy loading

## 🔧 Configuration

### Bridge Configuration

```yaml
bridge:
  host: "127.0.0.1"
  port: 8787
  max_sessions: 10
  session_ttl: 300
  allowed_roots: ["/workspace"]
```

### Plugin Configuration

```yaml
plugin:
  bridge_url: "ws://127.0.0.1:8787/ws"
  reconnect_interval: 5
  max_retries: 3
  heartbeat_interval: 30
```

## 🧪 Testing Strategy

### Unit Tests

- Component isolation
- Mock dependencies
- Edge case coverage
- Performance benchmarks

### Integration Tests

- End-to-end workflows
- Multi-component scenarios
- Error conditions
- Security validation

### Load Tests

- Concurrent sessions
- Message throughput
- Resource usage
- Failure recovery

## 📚 Monitoring & Observability

### Metrics

- Session count
- Command latency
- Error rates
- Resource usage

### Logging

- Structured logging
- Audit trails
- Error tracking
- Performance metrics

### Alerting

- Session failures
- High latency
- Resource exhaustion
- Security violations

---

**Arsitektur ini dirancang untuk memberikan kontrol sub-agent yang aman dan efisien di VoidEditor sambil menjaga keamanan dan auditabilitas yang ketat.**
