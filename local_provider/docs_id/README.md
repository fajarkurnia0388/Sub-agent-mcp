# MCP Local Provider — Model Agen AI Lokal

> **Bahasa**: [🇺🇸 English](../README.md) | [🇮🇩 Bahasa Indonesia](docs_id/README.md)

**Sistem provider lokal yang memungkinkan agen Cursor berperan sebagai model AI untuk VoidEditor**

## 🌟 Ringkasan

MCP Local Provider adalah sistem inovatif yang mengubah cara agen AI berinteraksi antar IDE. Alih-alih hanya mentransfer kontrol, sistem ini memungkinkan agen Cursor untuk berperan sebagai **model AI lokal** yang dapat diakses oleh VoidEditor, menciptakan kolaborasi AI antar editor yang mulus.

### Keunggulan Utama

✅ **Model AI Lokal** - Agen Cursor berfungsi sebagai provider model AI untuk VoidEditor  
✅ **Kompatibilitas API** - Mendukung endpoint OpenAI dan Ollama  
✅ **Sesi Aman** - Autentikasi berbasis token dan manajemen sesi efemeral  
✅ **Komunikasi Real-time** - Streaming respons dan komunikasi WebSocket  
✅ **Audit Lengkap** - Logging aktivitas dan keamanan  
✅ **Localhost Only** - Binding keamanan hanya ke 127.0.0.1

---

## 🏗️ Arsitektur Sistem

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│   VoidEditor    │◄──►│ Provider Adapter│◄──►│   Cursor IDE    │
│                 │    │   (FastAPI)     │    │   (Agen MCP)    │
│  Model Client   │    │                 │    │                 │
│                 │    │ 127.0.0.1:8788  │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                        │                        │
        │                        │                        │
        v                        v                        v
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│  Konfigurasi    │    │  Session Store  │    │   MCP Bridge    │
│     Model       │    │   (Memory)      │    │                 │
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Komponen Utama

1. **VoidEditor** - IDE yang menggunakan provider lokal sebagai model AI
2. **Provider Adapter** - Server FastAPI yang mengekspos API kompatibel OpenAI/Ollama
3. **MCP Bridge** - Sistem komunikasi dengan agen Cursor
4. **Cursor Agent** - Agen AI yang berperan sebagai model respons

---

## 🚀 Memulai Cepat

### Prasyarat

- Python 3.9+
- Cursor IDE dengan dukungan MCP
- VoidEditor dengan konfigurasi model lokal

### Instalasi

```bash
# Clone repository
git clone https://github.com/your-org/mcp-local-provider.git
cd mcp-local-provider

# Buat virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Salin konfigurasi template
cp config/config.yaml.example config/config.yaml

# Generate token keamanan
python tools/generate_token.py > .env
```

### Konfigurasi

Edit `config/config.yaml`:

```yaml
provider:
  host: "127.0.0.1"
  port: 8788
  model_name: "cursor-agent-local"

security:
  api_key: "${LOCAL_PROVIDER_TOKEN}"
  session_ttl: 300
  max_sessions: 5

mcp:
  bridge_url: "ws://127.0.0.1:8787"
  agent_id: "local-provider-1"
```

### Menjalankan Provider

```bash
# Start provider adapter
python -m local_provider.adapter

# Atau menggunakan uvicorn
uvicorn local_provider.adapter:app --host 127.0.0.1 --port 8788
```

### Konfigurasi VoidEditor

Tambahkan ke pengaturan VoidEditor:

```json
{
  "ai": {
    "provider": "openai",
    "apiUrl": "http://127.0.0.1:8788/v1",
    "apiKey": "your-generated-token",
    "model": "cursor-agent-local"
  }
}
```

---

## 📁 Struktur Proyek

```
local_provider/
├── README.md                 # Dokumentasi utama
├── docs_id/                  # Dokumentasi Indonesia
│   ├── README.md             # README Indonesia
│   ├── ARCHITECTURE.md       # Arsitektur Indonesia
│   ├── API_SPECIFICATION.md  # Spesifikasi API Indonesia
│   ├── DEPLOYMENT.md         # Panduan deployment Indonesia
│   ├── IMPLEMENTATION.md     # Panduan implementasi Indonesia
│   └── TESTING.md            # Panduan testing Indonesia
├── adapter_provider.py       # Server provider utama
├── models.py                 # Model data Pydantic
├── mcp_bridge.py            # Komunikasi MCP
├── config/                   # File konfigurasi
│   ├── config.yaml.example   # Template konfigurasi
│   └── security.yaml         # Konfigurasi keamanan
├── tools/                    # Utilitas
│   ├── generate_token.py     # Generator token
│   └── health_check.py       # Health check
├── tests/                    # Test suite lengkap
│   ├── unit/                 # Unit tests
│   ├── integration/          # Integration tests
│   ├── security/             # Security tests
│   └── e2e/                  # End-to-end tests
└── requirements.txt          # Dependencies Python
```

---

## 🔄 Alur Kerja Inti

### 1. Inisialisasi Provider

```python
# VoidEditor mengirim request model
POST /v1/chat/completions
{
  "model": "cursor-agent-local",
  "messages": [{"role": "user", "content": "Hello"}]
}
```

### 2. Komunikasi dengan Agen

```python
# Provider mengirim ke agen Cursor via MCP
{
  "type": "model_invoke",
  "id": "req-123",
  "model": "cursor-agent-local",
  "messages": [{"role": "user", "content": "Hello"}],
  "stream": true
}
```

### 3. Respons Streaming

```python
# Agen merespons via WebSocket
{
  "type": "model_stream_chunk",
  "id": "req-123",
  "chunk": {
    "choices": [{"delta": {"content": "Hello! How can I help you?"}}]
  }
}
```

### 4. Hasil ke VoidEditor

```javascript
// VoidEditor menerima streaming response
data: {"choices":[{"delta":{"content":"Hello! How can I help you?"}}]}
data: [DONE]
```

---

## 🔒 Fitur Keamanan

### Autentikasi API Key

- Token SHA-256 dengan panjang minimal 32 karakter
- Header `Authorization: Bearer` untuk semua request
- Token tidak disimpan dalam plain text

### Manajemen Sesi

- Sesi efemeral dengan TTL konfigurabel
- Cleanup otomatis sesi expired
- Rate limiting per sesi dan IP

### Isolasi Jaringan

- Binding hanya ke localhost (127.0.0.1)
- Firewall rules untuk membatasi akses
- TLS opsional untuk komunikasi internal

### Audit Logging

- Log semua request dan respons
- Tracking sesi dan autentikasi
- Monitoring aktivitas mencurigakan

---

## 📊 Persyaratan

### Minimum

| Komponen   | Requirement                                      |
| ---------- | ------------------------------------------------ |
| **OS**     | Windows 10/11, macOS 10.15+, Linux Ubuntu 20.04+ |
| **Python** | 3.9+                                             |
| **RAM**    | 256MB tersedia                                   |
| **Disk**   | 50MB ruang kosong                                |

### Direkomendasikan

| Komponen   | Requirement        |
| ---------- | ------------------ |
| **Python** | 3.11+              |
| **RAM**    | 512MB tersedia     |
| **Disk**   | 200MB ruang kosong |

### Dependencies

```txt
fastapi>=0.104.0
uvicorn>=0.24.0
websockets>=12.0
pydantic>=2.5.0
httpx>=0.25.0
pyyaml>=6.0
cryptography>=41.0.0
```

---

## 🔗 Dokumentasi Terkait

- **[ARCHITECTURE.md](docs_id/ARCHITECTURE.md)** - Arsitektur sistem detail
- **[API_SPECIFICATION.md](docs_id/API_SPECIFICATION.md)** - Spesifikasi API lengkap
- **[DEPLOYMENT.md](docs_id/DEPLOYMENT.md)** - Panduan deployment dan keamanan
- **[IMPLEMENTATION.md](docs_id/IMPLEMENTATION.md)** - Contoh kode implementasi
- **[TESTING.md](docs_id/TESTING.md)** - Strategi testing dan test cases

---

## 🤝 Kontribusi

Kami menyambut kontribusi! Silakan baca [CONTRIBUTING.md](CONTRIBUTING.md) untuk panduan kontribusi.

### Proses Development

1. Fork repository
2. Buat feature branch (`git checkout -b feature/amazing-feature`)
3. Commit perubahan (`git commit -m 'Add amazing feature'`)
4. Push ke branch (`git push origin feature/amazing-feature`)
5. Buka Pull Request

### Coding Standards

- Ikuti PEP 8 untuk Python code
- Tulis docstring untuk semua fungsi public
- Include type hints
- Tambahkan tests untuk fitur baru
- Update dokumentasi sesuai perubahan

---

## 📄 Lisensi

Proyek ini dilisensikan di bawah MIT License - lihat file [LICENSE](LICENSE) untuk detail.

---

## 🆘 Dukungan

### Mendapatkan Bantuan

- **Issues GitHub**: [Laporkan bug atau request fitur](https://github.com/your-org/mcp-local-provider/issues)
- **Diskusi**: [Komunitas dan Q&A](https://github.com/your-org/mcp-local-provider/discussions)
- **Email**: support@yourorg.com

### FAQ

**Q: Apakah sistem ini aman untuk penggunaan produksi?**  
A: Ya, dengan konfigurasi keamanan yang tepat. Sistem dirancang dengan prinsip security-first.

**Q: Bisakah menggunakan provider ini dengan IDE lain selain VoidEditor?**  
A: Ya, provider ini kompatibel dengan semua client yang mendukung OpenAI API.

**Q: Bagaimana cara mengkustomisasi respons model?**  
A: Anda dapat mengonfigurasi behavior melalui file konfigurasi atau mengimplementasikan custom handler.

---

**Dibuat dengan ❤️ untuk komunitas pengembang Indonesia**
