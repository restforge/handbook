# Protokol WebSocket

> Format URL koneksi, pesan client ke server, pesan server ke client, dan struktur broadcast message.

Kembali ke [ikhtisar Live Sync](README.md).

---

## Koneksi

URL koneksi WebSocket menggunakan format berikut:

```
ws://{SERVER_ADDRESS}:{LIVE_SYNC_PORT}/ws?x-api-key={API_KEY}
```

Contoh dengan REST API di port 3032 dan Live Sync di port 3033:

```
REST API  : http://192.168.1.100:3032/api/mini-inventory/category/create
WebSocket : ws://192.168.1.100:3033/ws?x-api-key=rk_abc123def456
```

API Key dikirim melalui query parameter `x-api-key`. Koneksi tanpa API Key yang valid ditolak dengan HTTP 401.

---

## Pesan Client ke Server

### Subscribe ke Channel

```json
{
  "type": "subscribe",
  "channels": ["mini-inventory:category"]
}
```

Subscribe ke beberapa channel sekaligus:

```json
{
  "type": "subscribe",
  "channels": ["mini-inventory:stock-inbound", "mini-inventory:stock-outbound"]
}
```

Subscribe dengan wildcard untuk menerima semua perubahan dalam satu project:

```json
{
  "type": "subscribe",
  "channels": ["mini-inventory:*"]
}
```

### Unsubscribe

```json
{
  "type": "unsubscribe",
  "channels": ["mini-inventory:category"]
}
```

### Pong (Respons Heartbeat)

```json
{
  "type": "pong"
}
```

Pong dikirim sebagai respons atas `ping` yang diterima dari server. Client yang tidak merespons dalam dua siklus heartbeat akan dihentikan secara otomatis.

---

## Pesan Server ke Client

### Konfirmasi Subscription

```json
{
  "type": "subscribed",
  "channels": ["mini-inventory:category"],
  "timestamp": "2026-04-12T10:30:00.000Z"
}
```

### Heartbeat Ping

Server mengirim ping setiap `LIVE_SYNC_HEARTBEAT_INTERVAL` detik (default 30).

```json
{
  "type": "ping",
  "timestamp": "2026-04-12T10:31:00.000Z"
}
```

### Notifikasi Data Change

```json
{
  "type": "data_change",
  "channel": "mini-inventory:category",
  "action": "create",
  "data": { ... },
  "requestId": "a1b2c3d4-e5f6-...",
  "timestamp": "2026-04-12T10:30:45.123Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `type` | string | Selalu `"data_change"` |
| `channel` | string | Channel spesifik yang memicu broadcast, mis. `"mini-inventory:category"` |
| `action` | string | `"create"`, `"update"`, atau `"delete"` |
| `data` | object/array | Isi berbeda per action — lihat detail di bawah |
| `requestId` | string/null | ID request dari client yang melakukan operasi. `null` jika module belum di-regenerate |
| `timestamp` | string | Waktu broadcast, format ISO 8601 |

---

## Struktur `data` per Action

### Action `create`

`data` berisi satu object — record yang baru dibuat dari hasil `RETURNING *` setelah INSERT.

```json
{
  "type": "data_change",
  "channel": "mini-inventory:category",
  "action": "create",
  "data": {
    "category_id": "a1b2c3d4-...",
    "category_code": "CAT007",
    "category_name": "Otomotif",
    "description": "Suku cadang kendaraan",
    "is_active": true
  },
  "requestId": "f8e7d6c5-...",
  "timestamp": "2026-04-13T10:30:45.123Z"
}
```

### Action `update`

`data` berisi satu object — record setelah di-update dari hasil `RETURNING *` setelah UPDATE.

```json
{
  "type": "data_change",
  "channel": "mini-inventory:category",
  "action": "update",
  "data": {
    "category_id": "a1b2c3d4-...",
    "category_code": "CAT001",
    "category_name": "Electronic Devices",
    "description": "Updated description",
    "is_active": true
  },
  "requestId": "f8e7d6c5-...",
  "timestamp": "2026-04-13T10:31:00.456Z"
}
```

### Action `delete`

`data` berisi array — satu atau lebih record yang dihapus, di-fetch sebelum DELETE.

```json
{
  "type": "data_change",
  "channel": "mini-inventory:category",
  "action": "delete",
  "data": [
    {
      "category_id": "a1b2c3d4-...",
      "category_code": "CAT006",
      "category_name": "Olahraga",
      "description": null,
      "is_active": false
    }
  ],
  "requestId": "f8e7d6c5-...",
  "timestamp": "2026-04-13T10:31:30.789Z"
}
```

> **Perlu diketahui:** Action `create` dan `update` mengirim `data` sebagai object, sedangkan `delete` mengirim `data` sebagai array. Perbedaan ini karena operasi delete dapat menghapus lebih dari satu record sekaligus via complex WHERE clause.

---

Lihat juga: [Integrasi Client-Side](client-integration.md) · [Konfigurasi](configuration.md)
