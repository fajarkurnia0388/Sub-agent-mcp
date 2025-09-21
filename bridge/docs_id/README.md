# MCP VoidEditor Bridge — README

> **Bahasa**: [🇺🇸 English](../README.md) | [🇮🇩 Bahasa Indonesia](README.md)

**Ringkasan**  
Proyek ini adalah _starter_ untuk membuat **MCP bridge** yang memungkinkan **sub-agen** beroperasi di dalam **VoidEditor** dengan kemampuan IDE lengkap, seperti agen bekerja di Cursor.

**Konsep**: Bayangkan Cursor sebagai **Kota Besar** tempat agen utama tinggal, dan VoidEditor sebagai **Kota Kecil** di dalam kota besar tempat sub-agen dapat bekerja secara independen. Pendekatan bridge menjaga VoidEditor tetap tidak dimodifikasi besar, memberi kontrol izin, dan memudahkan audit.

> Semua komponen dibuat untuk **lab / sandbox**. Terapkan hardening sebelum produksi.

---

## Komponen

1. **Bridge (FastAPI)** — `bridge_app.py`
   - Endpoint HTTP untuk: request access, approve/deny, session manajemen.
   - WebSocket untuk agent (session channel).
   - WebSocket atau SSE untuk notifikasi ke Cursor UI.
2. **VoidEditor Plugin (Sub-Agent Runtime)** — `plugin_void.js`
   - **Environment IDE Lengkap**: Menyediakan environment development lengkap untuk sub-agen.
   - Connects ke bridge (plugin WS) dan mengeksekusi operasi IDE lengkap.
   - Menangani: operasi file, analisis kode, terminal, search, git, manajemen proyek.
   - Sinkronisasi real-time: cursor, selection, viewport, buffers.
   - Jika VoidEditor tidak memiliki API, plugin bekerja melalui filesystem/CLI hooks.
3. **Cursor (client)** — konfigurasi MCP client di Cursor agar memanggil `POST /mcp/request_access` dan menampilkan UI consent.

---

## Fitur Sub-Agen yang Tersedia

**Operasi IDE Inti:**

- **Manajemen File**: Buat, baca, edit, hapus, rename file
- **Analisis Kode**: Analisis sintaks, deteksi error, code completion, formatting
- **Operasi Proyek**: Explore project tree, build, run tests, manajemen workspace
- **Integrasi Terminal**: Execute commands, manajemen sesi terminal
- **Version Control**: Operasi Git (status, commit, branch, diff)
- **Search & Replace**: Find in files, search project-wide, operasi regex
- **Sinkronisasi Real-time**: Posisi cursor, selection, viewport, perubahan buffer

**Manajemen Session:**

- Request access → user approve/deny via Cursor UI.
- Session ephemeral: token TTL (default 5 menit).
- WebSocket session proxy: sub-agen ↔ bridge ↔ plugin.
- Izin granular dengan kontrol akses berbasis scope.
- Root path restriction (ALLOWED_ROOT).

---

## Requirement (development)

- Python 3.10+
- Node.js 18+
- pip packages: `fastapi uvicorn python-multipart`
- npm packages: `ws` (atau sesuai plugin)

---

## Konfigurasi environment

Set environment variables (or gunakan `.env`):

- `MCP_TOKEN` — secret token untuk management endpoints (Cursor → bridge). **Jangan commit**.
- `ALLOWED_ROOT` — root workspace (contoh: `/home/minixo/lab/projects`)
- `HOST` — default `127.0.0.1`
- `PORT` — default `8787`
- `SESSION_TTL` — default `300` (detik)

---

## Instal & Jalankan (contoh)

1. Clone repo starter (mis. `starter-mcp-void`).
2. Virtual env Python:

```bash
python -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn
```

3. Export env var:

```bash
export MCP_TOKEN="dev-token"
export ALLOWED_ROOT="${HOME}/lab/projects"
```

4. Jalankan bridge:

```bash
uvicorn bridge_app:app --host 127.0.0.1 --port 8787 --reload
```

5. Jalankan plugin VoidEditor (contoh Node script):

```bash
node plugin_void.js
```

---

## Contoh alur (quick test)

1. Agent membuat request:

```bash
curl -X POST "http://127.0.0.1:8787/mcp/request_access" \
  -H "Authorization: ${MCP_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id":"agent-1",
    "scopes":["read:buffers","open:file"],
    "roots":["/home/minixo/lab/projects/myrepo"],
    "reason":"edit README"
  }'
```

2. Cursor UI menerima notifikasi `PENDING` (via SSE/WS), user klik **Approve**. Cursor memanggil:

```bash
curl -X POST "http://127.0.0.1:8787/mcp/approve" \
  -H "Authorization: ${MCP_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"request_id":"<id>","ttl_seconds":300}'
```

Response berisi `session_id` & `session_token`.

3. Agent connect ke WebSocket:

```text
ws://127.0.0.1:8787/mcp/session/<session_id>
Header: Authorization: Bearer <session_token>
```

4. Agent kirim pesan:

```json
{
  "type": "cmd",
  "id": "uuid-1",
  "action": "open_file",
  "args": { "path": "myrepo/README.md" }
}
```

Plugin di VoidEditor menerima, membuka file, kemudian mengirimkan `result`.

---

## Kebijakan & keamanan (penting)

- Bridge harus dijalankan di **localhost** atau Unix socket.
- Batasi `ALLOWED_ROOT`.
- Session token bersifat ephemeral (TTL).
- Batasi ukuran edit (`max_edit_bytes`) & rate limit.
- Audit semua permintaan (JSON log).
- Destructive actions memerlukan explicit confirmation UI.
- Awasi plugin untuk input validation (sanitize paths, no shell injection).

---

## Integrasi ke Cursor

Cursor perlu:

- Memanggil `POST /mcp/request_access` saat agen butuh akses.
- Mendengarkan notifikasi (SSE/WS) untuk menampilkan modal **Consent**.
- Memanggil `POST /mcp/approve` atau `POST /mcp/deny`.
- Menampilkan session status (time left) & tombol revoke.

---

## Logs & debugging

- Bridge mencetak audit JSON lines ke `./logs/audit.log`.
- Plugin mencetak event ke console.
- Gunakan Wireshark/tcpdump hanya di lab untuk inspect WebSocket traffic jika perlu.

---

## Next steps (opsional)

- Implement plugin yang menggunakan API internal VoidEditor (jika tersedia).
- Tambahkan persistent storage untuk requests & audit (sqlite).
- Tambah TLS / mTLS jika crossing host boundary.
- Implement CRDT/OT untuk multi-editor collaborative sessions (advanced).

---

## Dokumentasi

- **[BLUEPRINT.md](BLUEPRINT.md)** — Desain teknis lengkap (API, skema pesan, lifecycle, kebijakan keamanan, testing checklist, dan opsi integrasi ke Cursor & VoidEditor)
- **[../](../)** — Dokumentasi bahasa Inggris

---

**Butuh file implementasi?** Lihat [BLUEPRINT.md](BLUEPRINT.md) untuk spesifikasi teknis lengkap, atau versi bahasa Inggris di [../BLUEPRINT.md](../BLUEPRINT.md).
