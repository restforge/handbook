# Konfigurasi Live Sync

> Variable `.env`, arsitektur dedicated process, dan catatan teknis operasional.

Kembali ke [ikhtisar Live Sync](README.md).

---

## Variable `.env`

### Konfigurasi Minimal

```env
# Wajib: API Key authentication
KEY=rk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Live Sync
LIVE_SYNC_ENABLED=true
LIVE_SYNC_PORT=3033

# Redis (wajib untuk komunikasi antar process)
REDIS_HOST=localhost
REDIS_PORT=6380
```

### Konfigurasi Lengkap

```env
# API Key (wajib)
KEY=rk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Live Sync
LIVE_SYNC_ENABLED=true
LIVE_SYNC_PORT=3033
LIVE_SYNC_HEARTBEAT_INTERVAL=30

# Redis
REDIS_HOST=localhost
REDIS_PORT=6380
REDIS_PASSWORD=redis1234
REDIS_DB=0
```

### Referensi Parameter

| Variable | Tipe | Default | Keterangan |
|----------|------|---------|------------|
| `LIVE_SYNC_ENABLED` | boolean | `false` | Aktifkan Live Sync |
| `LIVE_SYNC_PORT` | number | wajib diisi | Port dedicated WebSocket server |
| `LIVE_SYNC_HEARTBEAT_INTERVAL` | number | `30` | Interval heartbeat dalam detik |

> **Perlu diketahui:** `LIVE_SYNC_PORT` harus berbeda dari `SERVER_PORT`. REST API dan Live Sync berjalan di port terpisah dari satu perintah `npx restforge serve`.

---

## Arsitektur Dedicated Process

Saat server dijalankan dengan `LIVE_SYNC_ENABLED=true`, RESTForge secara otomatis menjalankan dua process dari satu perintah:

```
npx restforge serve --project=mini-inventory --config=db-connection.env
    │
    │  server.js (supervisor)
    │
    ├─► Process 1: REST API (Express)
    │   Port: SERVER_PORT (mis. 3032)
    │   Tugas: handle HTTP request, CRUD, database
    │   Live Sync role: publisher-only (publish event ke Redis)
    │
    └─► Process 2: Live Sync Server (Dedicated WebSocket)
        Port: LIVE_SYNC_PORT (mis. 3033)
        Tugas: manage WebSocket connections, heartbeat, broadcast
        Live Sync role: full (subscribe Redis, broadcast ke clients)
```

Pemisahan ini memberikan tiga manfaat:

- Event loop REST API tidak terganggu oleh persistent WebSocket connections dan heartbeat.
- Jika Live Sync process crash, REST API tetap melayani request HTTP secara normal.
- Heap memory untuk WebSocket connections tidak berkompetisi dengan REST API.

Pengguna tidak perlu menjalankan dua perintah terpisah. Supervisor mengelola kedua process secara transparan.

---

## Single Mode vs Cluster Mode

| Mode | REST API | Live Sync | Komunikasi |
|------|----------|-----------|------------|
| Single | 1 process | 1 dedicated process | Redis pub/sub |
| Cluster (mis. 4 worker) | 4 worker process | 1 dedicated process (di-spawn oleh master) | Redis pub/sub |

Dalam kedua mode, hanya ada satu Live Sync process yang mengelola semua koneksi WebSocket. Setiap REST API worker menggunakan broadcaster dalam mode publisher-only.

---

## Health Check

Live Sync process menyediakan endpoint untuk memantau status koneksi.

```
GET http://localhost:{LIVE_SYNC_PORT}/health
```

```json
{
  "status": "healthy",
  "service": "livesync",
  "port": 3033,
  "clients": 12,
  "channels": 3,
  "uptime": 3600.5,
  "timestamp": "2026-04-13T10:30:00.000Z"
}
```

---

## Catatan Teknis

### Urutan Eksekusi Broadcast

Broadcast berjalan setelah COMMIT selesai, sebagai fire-and-forget yang tidak memblokir HTTP response.

```
REST API Process:
  1. Database operation (INSERT/UPDATE/DELETE)
  2. onAfter events (jika ada Component Engine)
  3. COMMIT transaction
  4. invalidateCache()
  5. publish event ke Redis   ← publisher-only (~1ms)
  6. return HTTP response

Live Sync Process (async, terpisah):
  7. subscribe dari Redis → terima event
  8. broadcast ke WebSocket clients
```

Kegagalan broadcast (error WebSocket, client disconnect) tidak mempengaruhi response CRUD. Semua error di-log sebagai warning tanpa throw.

### Dead Connection Handling

Server mengirim `ping` setiap `LIVE_SYNC_HEARTBEAT_INTERVAL` detik. Client yang tidak merespons `pong` dalam dua siklus berturut-turut dianggap mati dan dihentikan secara otomatis. Semua channel subscription untuk koneksi tersebut dibersihkan.

### Graceful Shutdown

Saat server menerima SIGTERM atau SIGINT, kedua process dimatikan secara berurutan.

**REST API process (supervisor):**
1. Tutup Redis connection publisher.
2. Kirim SIGTERM ke Live Sync child process.
3. Grace period untuk operasi HTTP yang masih berjalan.

**Live Sync process (child):**
1. Semua WebSocket client menerima close frame dengan kode 1001 (Going Away).
2. WebSocket server ditutup.
3. Redis subscriber di-unsubscribe dan diputus.
4. HTTP server health check ditutup.

### Konvensi Nama Channel

Channel menggunakan format `{project}:{endpoint}` yang diambil dari `modelMetadata` di setiap generated model. Format ini konsisten dengan pola URL API.

Contoh: endpoint yang dibuat dengan `--project=mini-inventory --endpoint=category` menghasilkan channel `mini-inventory:category`, sama dengan prefix URL `/api/mini-inventory/category/...`.

---

Lihat juga: [Protokol WebSocket](protocol.md) · [Integrasi Client-Side](client-integration.md)
