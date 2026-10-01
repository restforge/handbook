# Payload Dashboard

Payload dashboard berfungsi untuk mendefinisikan kumpulan widget SQL aggregasi beserta kontrak parameter yang berlaku untuk seluruh widget. Format payload berbeda dari payload CRUD karena tidak mengandung `tableName`, field validation, atau action.

---

## Struktur Dasar

```json
{
  "params": {
    "from":    { "type": "date",   "required": true  },
    "to":      { "type": "date",   "required": true  },
    "country": { "type": "string", "required": false, "default": "ALL" }
  },
  "widgets": [
    {
      "id": "author_sales",
      "query": "file:query/dashboard-sales/author-sales.sql"
    },
    {
      "id": "expected_earnings",
      "queries": {
        "value": "file:query/dashboard-sales/earnings-value.sql",
        "trend": "file:query/dashboard-sales/earnings-trend.sql",
        "items": "file:query/dashboard-sales/earnings-breakdown.sql"
      }
    }
  ]
}
```

Validator membedakan payload CRUD dan dashboard berdasarkan keberadaan `tableName` versus `widgets`:

| Ada field | Tipe payload |
|-----------|--------------|
| `tableName` | CRUD |
| `widgets` | Dashboard |

---

## Referensi Field Payload

| Field | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| `widgets` | array | ya | Minimal 1 widget |
| `widgets[].id` | string | ya | Unique antar widget; menjadi key di response |
| `widgets[].query` | string | kondisional | File reference `file:path/to.sql`; mutually exclusive dengan `queries` |
| `widgets[].queries` | object | kondisional | Map key ke file reference; minimal 1 key; mutually exclusive dengan `query` |
| `params` | object | tidak | Kontrak parameter berlaku untuk seluruh widget |
| `params[k].type` | string | ya | Enum: `string`, `number`, `boolean`, `date` |
| `params[k].required` | boolean | tidak | Default `false` |
| `params[k].default` | any | tidak | Nilai default bila request tidak kirim param |
| `cache` | object | tidak | Konfigurasi Redis cache opsional |

---

## Mendefinisikan Parameter Request

Blok `params` mendefinisikan kontrak parameter yang dikirim via request body dan di-bind ke placeholder SQL di seluruh widget. Runtime memvalidasi tipe dan field `required` sebelum eksekusi.

```json
{
  "params": {
    "from":    { "type": "date",   "required": true  },
    "to":      { "type": "date",   "required": true  },
    "country": { "type": "string", "required": false, "default": "ALL" }
  }
}
```

SQL yang menggunakan parameter ini memakai placeholder `:name`:

```sql
SELECT category, COALESCE(SUM(amount), 0) AS value
FROM sales
WHERE country = :country
  AND sale_date BETWEEN :from AND :to
GROUP BY category;
```

> **Perlu diketahui:** Runtime mensubstitusi `:name` ke bind syntax dialect target secara otomatis — `$1`, `$2` untuk PostgreSQL; `?` untuk MySQL; `:1`, `:2` untuk Oracle. Detail di [Multi-Dialect Support](generate.md#dukungan-multi-dialect-multi-dialect-support).

Validator memindai setiap SQL widget dan throw error bila ada placeholder yang tidak terdaftar di `params`. Ini mencegah unbound placeholder saat runtime.

---

## Field Terlarang

Validator menolak field berikut karena menyatukan concern backend dan frontend:

| Field terlarang | Alasan |
|-----------------|--------|
| `widgetType` (donut, bar, pie, area) | Pilihan visual frontend |
| `layout` | Variant layout frontend |
| `title`, `subtitle` | Label UI terkait i18n |
| `color` | Hex color untuk theming |

Field-field ini ditolak baik di root payload maupun di level widget.

---

## Mengaktifkan Cache

Blok `cache` opsional mengaktifkan Redis-based caching untuk seluruh response envelope dashboard.

```json
{
  "cache": {
    "enabled": true,
    "ttl": 300,
    "invalidates": ["stock_inbound", "supplier"]
  }
}
```

| Field | Tipe | Wajib | Default | Keterangan |
|-------|------|-------|---------|------------|
| `cache.enabled` | boolean | ya | — | Toggle cache untuk dashboard ini |
| `cache.ttl` | number | tidak | inherit `CACHE_TTL` env | TTL dalam detik; `0` disable cache entry tersebut |
| `cache.invalidates` | array<string> | tidak | `[]` | Nama tabel yang memicu cascade invalidation saat ada WRITE |

> **Perlu diketahui:** Cache scope adalah full response envelope per kombinasi `params` dan subset `widgets[]`. Cascade invalidation otomatis terpicu saat ada operasi WRITE (create/update/delete) pada tabel yang terdaftar di `invalidates`.

### Cara Cache Invalidation Bekerja

```
WRITE pada tabel "stock_inbound"
    ↓
base-model.invalidateCache()
    ↓
registry lookup → ['rf:project:dash-inbound:dashboard:*']
    ↓
Redis DEL semua key match
    ↓
Request berikutnya: MISS → eksekusi SQL → SET cache key baru
```

### Validasi Cross-Reference Cache

Generator memvalidasi konsistensi `cache.invalidates` dengan tabel yang muncul di SQL widget saat generate time.

| Skenario | Severity |
|----------|----------|
| Tabel terdeteksi di SQL widget dan terdaftar di metadata project, tapi tidak ada di `invalidates` | Error stop |
| Tabel ada di `invalidates` tapi tidak muncul di SQL widget manapun | Error stop |
| Tabel ada di SQL widget tapi tidak terdaftar di metadata project | Warning (kemungkinan view, CTE alias, atau cross-project table) |

Setelah mengedit SQL widget yang menambah JOIN ke tabel baru, perbarui `invalidates` di payload sebelum menjalankan ulang generate.

---

## Validasi Payload

Validator dijalankan secara otomatis sebelum generator menulis file. Aturan utama:

**Root level:**
- `widgets` wajib array dengan minimal 1 entry
- `tableName` tidak boleh ada (mutually exclusive dengan `widgets`)
- Field terlarang tidak boleh ada

**Per widget:**
- `id` wajib string non-empty dan unique antar widget
- Tepat satu dari `query` atau `queries` harus ada; keduanya sekaligus atau tidak ada keduanya menyebabkan error
- `queries` wajib minimal 1 key; setiap value bertipe string

**Params contract:**
- `params[k].type` wajib ada dan termasuk `string`, `number`, `boolean`, `date`
- `params[k].required` (jika ada) wajib boolean
- Setiap placeholder `:name` di SQL wajib terdaftar di `params`

**Cache (jika ada):**
- `cache.enabled` wajib boolean
- Tabel di `invalidates` wajib konsisten dengan SQL widget dan metadata project
