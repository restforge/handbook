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

Dropdown filter untuk SATU field "status-like" — slot utama toolbar, bukan khusus
field boolean. Field status utama bisa:

- **Boolean** (mis. `is_active`) → `options` berisi `true`/`false` dengan label
  Active/Inactive.
- **Enum** (mis. `status` dengan `constraints.enum` di RDF, 3+ nilai) → `options`
  berisi nilai enum apa adanya, tanpa konversi ke Active/Inactive.

Contoh boolean:

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

Contoh enum (field `status` 3 nilai):

```json
"features": {
    "enableStatusFilter": true,
    "statusFilter": {
        "field": "status",
        "label": "Status",
        "options": [
            {"value": "waiting", "text": "Waiting"},
            {"value": "called", "text": "Called"},
            {"value": "done", "text": "Done"}
        ]
    }
}
```

| Properti `statusFilter` | Tipe | Wajib | Keterangan |
|-------------------------|------|:-----:|-----------|
| `field` | string | ✓ | Nama field yang difilter (`is_active` atau `status`) |
| `label` | string | ✓ | Label dropdown di toolbar |
| `options` | array | ✓ | Array `{value, text}` sebagai pilihan filter. Boolean: `true`/`false` Active/Inactive. Enum: nilai enum RDF apa adanya |

Behavior:

| Aspek | Behavior |
|-------|----------|
| Default | Dropdown menampilkan opsi "All" (tanpa filter) |
| Perubahan pilihan | DataTable di-reload dengan kondisi filter |
| Payload ke backend | Dikirim dalam array `where` sebagai `{ "key": "<field>", "value": "<selected>" }` |

### Auto-generation via `payload migrate`

Sejak `payload migrate` (backend), `enableStatusFilter`/`statusFilter` TIDAK LAGI
perlu ditulis manual bila RDF punya field bernama `is_active` atau `status` yang
beresolusi boolean (`fieldType: checkbox`) atau select+enum
(`fieldType: select`, `extra.dataSource.type: static`). Migrator otomatis memilih
field pertama (urutan `fieldName` RDF) yang cocok sebagai field status utama dan
menghasilkan `statusFilter` dengan `options` sesuai tipenya (lihat dua contoh di
atas). Field `is_active` (khusus boolean) JUGA otomatis mendapat `defaultScope` di
level RDF — perilaku ini sudah ada sebelum auto-generation filter dan tidak
berubah.

UDF hasil `payload migrate` tetap boleh diedit manual (mis. ubah `label`, susun
ulang `options`) — migrator hanya mengisi nilai awal, tidak mengunci field ini.

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
| `field` | string | ✓ | Nama kolom DB aktual untuk filter ini. WAJIB diisi untuk plugin `vanilla-js-custom` dan `vanilla-js-auth` (filter tanpa `field` di-skip diam-diam oleh kedua plugin tersebut). Diabaikan oleh `vanilla-js-basic`. Disarankan SELALU sama dengan `name` kecuali ada kebutuhan aliasing eksplisit |
| `label` | string | ✓ | Label dropdown di toolbar |
| `dataSource` | object | ✓ | Sumber data filter. Struktur sama dengan `dataSource` field `select` |

Lihat [`data-source.md`](./data-source.md) untuk detail struktur `dataSource`.

### Auto-generation via `payload migrate`

Sejak `payload migrate` (backend), `enableDataFilter`/`dataFilters[]` otomatis
digenerate untuk:

1. **Field FK (JOIN)** — field yang beresolusi `fieldType: select` dengan
   `extra.dataSource.type: api` (lihat [`FK + JOIN menjadi select`](../../commands/restforge-backend/payload/migrate.md#fk--join-menjadi-select)).
2. **Field enum non-status** — field yang beresolusi `fieldType: select` dengan
   `extra.dataSource.type: static`, KECUALI field tersebut sudah terpilih sebagai
   field status utama (lihat [`statusFilter`](#enablestatusfilter-dan-statusfilter)
   di atas).

Setiap entri yang digenerate otomatis selalu punya `field === name` — migrator
tidak punya sumber data lain untuk aliasing kolom. Contoh (RDF field `priority`
enum + `category_id` FK):

```json
"dataFilters": [
    {
        "name": "priority",
        "field": "priority",
        "label": "Priority",
        "dataSource": {
            "type": "static",
            "options": [
                {"value": "low", "text": "Low"},
                {"value": "medium", "text": "Medium"},
                {"value": "high", "text": "High"}
            ]
        }
    },
    {
        "name": "category_id",
        "field": "category_id",
        "label": "Category",
        "dataSource": {
            "type": "api",
            "resource": "categories",
            "select": ["id", "category_name"]
        }
    }
]
```

Urutan entri mengikuti urutan `fieldName` di RDF. UDF hasil migrate tetap boleh
diedit manual (mis. set `display` eksplisit, lihat
[Maks 2 Filter Inline + Filter Button](#maks-2-filter-inline--filter-button) di
bawah).

### Urutan Inisialisasi

Filter bertipe `api` memuat data secara asinkron via AJAX. Generator memastikan semua filter API selesai dimuat sebelum DataTable diinisialisasi melalui counter-based callback pattern. Tidak diperlukan konfigurasi tambahan untuk menangani race condition.

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

### Maks 2 Filter Inline + Filter Button

Toolbar datagrid menampilkan maksimal **2 filter inline** (gabungan `statusFilter`
+ `dataFilters[]`). Selebihnya otomatis masuk **"filter button"** (tombol
"More Filters" / panel dropdown), bukan disembunyikan.

Urutan slot deterministik:

1. `statusFilter` (bila `enableStatusFilter: true`) — SELALU mengambil slot
   inline pertama.
2. Sisa slot diisi dari `dataFilters[]` sesuai urutan array.
3. Filter ke-3 dan seterusnya (dari gabungan `statusFilter` + `dataFilters[]`)
   masuk filter button.

| Skenario | `statusFilter`? | `dataFilters[]` | Inline (maks 2) | Filter button |
|---|---|---|---|---|
| A | ada | `[fk_a, enum_b]` | `statusFilter`, `fk_a` | `enum_b` |
| B | tidak ada | `[fk_a, enum_b, enum_c]` | `fk_a`, `enum_b` | `enum_c` |
| C | ada | `[fk_a]` | `statusFilter`, `fk_a` | (kosong, tombol tidak muncul) |

**Override manual menang.** Bila `dataFilters[].display` di-set eksplisit
(`"inline"` atau `"dropdown"`) — baik ditulis tangan maupun hasil edit manual
setelah migrate — nilai itu dipakai apa adanya dan TIDAK ikut dihitung kuota 2
slot otomatis. Filter lain (tanpa `display` eksplisit) tetap dihitung otomatis
dari urutan dan sisa kuota. Konsekuensinya, filter manual `"inline"` BISA membuat
total filter inline melebihi 2 bila semua filter lain juga di-set manual
`"inline"`.

```json
"dataFilters": [
    {
        "name": "supplier_id",
        "field": "supplier_id",
        "label": "Supplier",
        "display": "dropdown",
        "dataSource": { "type": "api", "resource": "suppliers", "select": ["id", "supplier_name"] }
    },
    {
        "name": "warehouse_id",
        "field": "warehouse_id",
        "label": "Warehouse",
        "dataSource": { "type": "api", "resource": "warehouses", "select": ["id", "warehouse_name"] }
    }
]
```

Pada contoh di atas, `supplier_id` SELALU di filter button (override eksplisit),
sedangkan `warehouse_id` dihitung otomatis dari sisa kuota (tidak terganggu oleh
override `supplier_id`).

**Perbedaan perilaku antar plugin** (tidak identik, perlu diperhatikan saat pilih
`--plugin`):

| Plugin | Mekanisme filter button |
|--------|--------------------------|
| `vanilla-js-basic`, `vanilla-js-custom` | Tombol + dropdown panel custom (markup, CSS, dan JS toggle baru — tidak memakai Bootstrap) |
| `vanilla-js-auth` (Metronic) | Memakai mekanisme dropdown filter native yang sudah ada sebelumnya (`data-kt-menu-trigger`/Metronic KTMenu). Tidak ada markup baru — hanya nilai `display` per filter yang sekarang dihitung otomatis dari slot, bukan lagi sekadar dibaca apa adanya dari payload |

`statusFilter` SELALU dianggap inline (tidak pernah masuk filter button) — tidak
ada properti `statusFilter.display` yang relevan untuk override slot, karena
slotnya tidak pernah dihitung dari kuota.

## `enableLiveSync`

Mengaktifkan WebSocket Live Sync untuk halaman ini. Tabel data akan di-refresh secara otomatis ketika ada perubahan dari client lain.

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

Jika `fieldLayout: "horizontal"`, blok `fieldRows` diabaikan (validator menghasilkan warning). Layout horizontal selalu single-column.

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
2. Filter data 1, 2, ... (jika aktif, maksimal 2 filter gabungan tampil inline —
   lihat [Maks 2 Filter Inline + Filter Button](#maks-2-filter-inline--filter-button))
3. Search box (jika aktif)
4. Tombol aksi (Add, Refresh, dst.)

---

← [`field-rows.md`](./field-rows.md) | [Selanjutnya: `master-detail.md`](./master-detail.md) →
