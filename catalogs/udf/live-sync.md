# Live Sync via WebSocket — `liveSync`

> Konfigurasi WebSocket untuk auto-refresh tabel data ketika ada perubahan dari client lain.

## Konsep

Live Sync menghubungkan aplikasi frontend ke server WebSocket. Ketika ada event perubahan data dari client lain (mis. user lain men-save record), tabel data otomatis di-reload tanpa user perlu klik refresh manual.

Konfigurasi terdiri dari dua bagian:

1. **Blok `liveSync` di level root** — endpoint WebSocket dan API key
2. **`features.enableLiveSync: true` di setiap page** yang ingin menerima event

Kedua bagian harus konsisten. Tidak boleh satu page mengaktifkan `enableLiveSync` tanpa blok root, dan sebaliknya.

## Sintaks Blok Root

```json
{
    "liveSync": {
        "url": "wss://sync.example.com/ws",
        "apiKey": "live-sync-api-key-here"
    },
    "pages": [
        {
            "pageId": "customer",
            "features": {
                "enableLiveSync": true
            },
            /* ... */
        }
    ]
}
```

## Properti `liveSync`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `url` | string | ✓ | URL WebSocket. Harus berawalan `ws://` (insecure) atau `wss://` (secure) |
| `apiKey` | string | ✓ | API key untuk autentikasi koneksi WebSocket |

## Aturan Validasi

### Blok Root

| Aturan | Hasil |
|--------|-------|
| `liveSync.url` non-empty | Error: `"liveSync.url must be a non-empty string"` |
| `liveSync.url` scheme valid | Error: `"liveSync.url must start with 'ws://' or 'wss://', got: '<value>'"` |
| `liveSync.apiKey` non-empty | Error: `"liveSync.apiKey must be a non-empty string"` |

### Konsistensi dengan Page

| Kondisi | Hasil |
|---------|-------|
| Blok `liveSync` ada, tidak ada page dengan `features.enableLiveSync: true` | Warning: `"liveSync block is defined but no page has features.enableLiveSync enabled"` |
| Page dengan `features.enableLiveSync: true`, tidak ada blok `liveSync` di root | Error per page: `"[<pageId>] features.enableLiveSync is true but top-level 'liveSync' block is not defined"` |

## Skema URL

| Scheme | Konteks Pemakaian |
|--------|-------------------|
| `ws://` | Development atau internal network. Tidak encrypt traffic |
| `wss://` | Production. WebSocket over TLS, encrypt traffic |

Validator menolak scheme lain (`http://`, `https://`, `tcp://`, dll.) karena WebSocket browser hanya mendukung `ws://` dan `wss://`.

## Mengaktifkan per Page

Tidak semua page perlu Live Sync. Aktifkan hanya di page yang membutuhkan auto-refresh:

```json
{
    "liveSync": {
        "url": "wss://sync.example.com/ws",
        "apiKey": "key"
    },
    "pages": [
        {
            "pageId": "stock-balance",
            "features": {"enableLiveSync": true},
            /* ... live data, butuh sync ... */
        },
        {
            "pageId": "master-product",
            "features": {"enableLiveSync": false},
            /* ... master data, jarang berubah, tidak butuh sync ... */
        }
    ]
}
```

## Implementasi Runtime

Implementasi WebSocket client di JavaScript yang di-generate ditangani oleh plugin. Behavior umum:

1. Saat halaman load, JavaScript membuka koneksi WebSocket ke `liveSync.url` dengan header autentikasi `apiKey`
2. Server WebSocket mengirim event JSON ketika ada perubahan data di resource yang berkaitan dengan page
3. JavaScript menerima event, memvalidasi `pageId`, dan trigger reload DataTable

Detail format event, channel subscription, dan reconnect logic bergantung pada plugin dan server WebSocket. Lihat dokumentasi plugin untuk spesifikasi format event yang didukung.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Data sering berubah lintas user (stock, queue) | Aktifkan Live Sync |
| Master data jarang berubah (kategori, region) | Tidak perlu Live Sync |
| Dashboard real-time | Live Sync atau polling tergantung kebutuhan |
| Aplikasi internal tanpa multi-user | Tidak perlu Live Sync |
| URL production | Selalu pakai `wss://`, jangan `ws://` |

## Contoh Lengkap

```json
{
    "appConfig": {
        "appName": "Warehouse Management",
        "appCode": "wms",
        "plugin": "vanilla-js-auth",
        "apiBaseUrl": "https://api.example.com/wms",
        "port": 3000
    },
    "liveSync": {
        "url": "wss://sync.example.com/ws",
        "apiKey": "wms-live-sync-key-prod"
    },
    "pages": [
        {
            "pageId": "stock-balance",
            "pageTitle": "Stock Balance",
            "apiPath": "stock-balance",
            "primaryKey": "stock_id",
            "displayField": "item_name",
            "features": {
                "enableSearch": true,
                "enableLiveSync": true
            },
            "fields": [
                {"name": "item_name", "label": "Item", "type": "text", "inTable": true, "tableOrder": 1},
                {"name": "warehouse_name", "label": "Gudang", "type": "text", "inTable": true, "tableOrder": 2},
                {"name": "balance_qty", "label": "Stok", "type": "number", "inTable": true, "tableOrder": 3}
            ]
        }
    ]
}
```

---

← [`homepage.md`](./homepage.md) | [Selanjutnya: `dashboard-page.md`](./dashboard-page.md) →
