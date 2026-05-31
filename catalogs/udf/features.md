# Fitur Halaman — `features`

> Konfigurasi fitur yang aktif di toolbar dan form per page CRUD.

## Sintaks

```json
"features": {
    "enableSearch": true,
    "enableStatusFilter": true,
    "statusFilter": { /* ... */ },
    "enableDataFilter": true,
    "dataFilters": [ /* ... */ ],
    "enableLiveSync": false,
    "fieldLayout": "vertical"
}
```

## Properti Top-Level

| Properti | Tipe | Default | Keterangan |
|----------|------|---------|-----------|
| `enableSearch` | boolean | `false` | Menampilkan search box di toolbar |
| `enableStatusFilter` | boolean | `false` | Menampilkan dropdown filter status di toolbar |
| `statusFilter` | object | — | Konfigurasi filter status (wajib jika `enableStatusFilter: true`) |
| `enableDataFilter` | boolean | `false` | Menampilkan satu/lebih dropdown filter data di toolbar |
| `dataFilters` | array | — | Konfigurasi filter data (wajib jika `enableDataFilter: true`) |
| `enableLiveSync` | boolean | `false` | Mengaktifkan WebSocket live sync (memerlukan blok `liveSync` di level root) |
| `fieldLayout` | string | `"vertical"` | Orientasi label: `"vertical"` atau `"horizontal"` |

## `enableSearch`

Search box di toolbar untuk pencarian teks di tabel data.

```json
"features": {
    "enableSearch": true
}
```

Behavior:

- Toolbar menampilkan input text dengan tombol clear
- Pencarian bersifat server-side: nilai search dikirim ke backend sebagai parameter `search.value` dalam request DataTable

## `enableStatusFilter` dan `statusFilter`

Dropdown filter status untuk memfilter data berdasarkan field boolean (mis. Active/Inactive).

```json
"features": {
    "enableStatusFilter": true,
    "statusFilter": {
        "field": "is_active",
        "label": "Status",
        "options": [
            {"value": "true", "text": "Active"},
            {"value": "false", "text": "Inactive"}
        ]
    }
}
```

| Properti `statusFilter` | Tipe | Wajib | Keterangan |
|-------------------------|------|:-----:|-----------|
| `field` | string | ✓ | Nama field yang difilter (biasanya `is_active`) |
| `label` | string | ✓ | Label dropdown di toolbar |
| `options` | array | ✓ | Array `{value, text}` sebagai pilihan filter |

Behavior:

| Aspek | Behavior |
|-------|----------|
| Default | Dropdown menampilkan opsi "All" (tanpa filter) |
| Perubahan pilihan | DataTable di-reload dengan kondisi filter |
| Payload ke backend | Dikirim dalam array `where` sebagai `{ "key": "<field>", "value": "<selected>" }` |

## `enableDataFilter` dan `dataFilters`

Satu atau lebih dropdown filter di toolbar berdasarkan field tertentu. Mendukung sumber data `static` dan `api` (mirip field `select`).

```json
"features": {
    "enableDataFilter": true,
    "dataFilters": [
        {
            "name": "city_id",
            "label": "Kota",
            "dataSource": {
                "type": "api",
                "resource": "city",
                "select": ["city_id", "city_name"]
            }
        },
        {
            "name": "contact_type",
            "label": "Tipe Kontak",
            "dataSource": {
                "type": "static",
                "options": [
                    {"value": "personal", "text": "Personal"},
                    {"value": "business", "text": "Business"}
                ]
            }
        }
    ]
}
```

### Properti `dataFilters[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `name` | string | ✓ | Nama field untuk key di array `where` yang dikirim ke backend |
| `label` | string | ✓ | Label dropdown di toolbar |
| `dataSource` | object | ✓ | Sumber data filter. Struktur sama dengan `dataSource` field `select` |

Lihat [`data-source.md`](./data-source.md) untuk detail struktur `dataSource`.

### Urutan Inisialisasi

Filter bertipe `api` memuat data asynchronous via AJAX. Generator memastikan semua filter API selesai dimuat sebelum DataTable diinisialisasi melalui counter-based callback pattern. Tidak diperlukan konfigurasi tambahan untuk menangani race condition.

### Koeksistensi dengan `enableStatusFilter`

`enableStatusFilter` dan `enableDataFilter` bersifat independen — keduanya dapat aktif bersamaan. Semua filter dikumpulkan ke satu array `where` yang dikirim ke backend:

```json
{
    "where": [
        {"key": "is_active", "value": "true"},
        {"key": "city_id", "value": "ct-xxxxx"},
        {"key": "contact_type", "value": "business"}
    ]
}
```

Backend memproses array `where` dengan logika AND (semua kondisi harus terpenuhi).

## `enableLiveSync`

Mengaktifkan WebSocket Live Sync untuk halaman ini. Tabel data akan otomatis refresh ketika ada perubahan dari client lain.

```json
"features": {
    "enableLiveSync": true
}
```

Aktivasi per-page membutuhkan blok `liveSync` di level root payload. Tanpa blok root, validator menghasilkan error:

```
[<pageId>] features.enableLiveSync is true but top-level 'liveSync' block is not defined
```

Lihat [`live-sync.md`](./live-sync.md) untuk konfigurasi blok root `liveSync`.

## `fieldLayout`

Orientasi label relatif terhadap input field.

```json
"features": {
    "fieldLayout": "vertical"
}
```

| Nilai | Behavior |
|-------|----------|
| `"vertical"` (default) | Label di atas input. Setiap field menempati satu baris penuh |
| `"horizontal"` | Label di samping input. Label dan input ditempatkan sebaris |

### Visualisasi

**Vertical (default):**

```
Nama Kontak
[___________________]

Email
[___________________]
```

**Horizontal:**

```
Nama Kontak  [___________________]
Email        [___________________]
```

### Interaksi dengan `fieldRows`

Jika `fieldLayout: "horizontal"`, blok `fieldRows` di-ignore (validator menghasilkan warning). Layout horizontal selalu single-column.

## Kombinasi Fitur

Semua flag fitur dapat dikombinasikan bebas:

```json
"features": {
    "enableSearch": true,
    "enableStatusFilter": true,
    "statusFilter": { /* ... */ },
    "enableDataFilter": true,
    "dataFilters": [ /* ... */ ],
    "enableLiveSync": true,
    "fieldLayout": "vertical"
}
```

Toolbar akan menampilkan (dari kiri ke kanan):
1. Filter status (jika aktif)
2. Filter data 1, 2, ... (jika aktif)
3. Search box (jika aktif)
4. Tombol aksi (Add, Refresh, dst.)

---

← [`field-rows.md`](./field-rows.md) | [Selanjutnya: `master-detail.md`](./master-detail.md) →
