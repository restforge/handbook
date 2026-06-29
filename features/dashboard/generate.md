# Generate & Best Practices

---

## Membuat Endpoint Dashboard

Subcommand `dashboard create` menghasilkan satu file handler di `src/modules/{project}/{name}.js` dengan semua SQL widget sudah di-embed sebagai template literal.

```bash
npx restforge dashboard create \
  --project=<project-name> \
  --name=<dashboard-name> \
  --payload=<payload-file>
```

### Argumen CLI

| Argumen | Wajib | Default | Keterangan |
|---------|-------|---------|------------|
| `--project=<name>` | ya | — | Nama project target |
| `--name=<name>` | ya | — | Nama dashboard; wajib berawalan `dash-` (misalnya `dash-sales`, `dash-inbound`) |
| `--payload=<file>` | ya | — | Nama file payload JSON di folder `payload/` (dengan atau tanpa ekstensi `.json`) |
| `--database=<type>` | tidak | `postgres` | Dialect database: `postgres`, `mysql`, `oracle` |
| `--force=<bool>` | tidak | `false` | `true` untuk overwrite file existing (file lama di-archive) |

> **Perlu diketahui:** Nilai `--name` menjadi filename `{name}.js` dan URL segment. Validator menolak nama tanpa prefix `dash-` dengan pesan error yang jelas.

### Contoh Output

```
Processing payload...
Payload processing completed

Generating main module: mini-inventory (postgres)
Main module mini-inventory.js generated successfully

Dashboard module created: src/modules/mini-inventory/dash-sales.js

URL: POST /api/mini-inventory/dash-sales/dashboard
```

---

## Menyiapkan Project

### Prasyarat

1. Project sudah diinisialisasi via `npx restforge init` sehingga mempunyai struktur folder `payload/`, `src/`, `metadata/`, `config/`.
2. Database connection sudah dikonfigurasi di `config/db-connection.env`.
3. Tabel sumber data sudah tersedia di database.

### Struktur Folder

Organisasikan payload dan SQL dalam sub-folder agar tidak bercampur dengan asset CRUD:

```
payload/
├── dashboard-sales.json              ← payload dashboard
└── query/
    └── dashboard-sales/              ← sub-folder per dashboard
        ├── author-sales.sql
        ├── sales-this-months-value.sql
        └── sales-this-months-points.sql
```

Konvensi `query/dashboard-{name}/` mencegah file SQL dashboard bercampur dengan SQL CRUD dan memudahkan identifikasi file SQL milik dashboard mana.

### Format File Reference

Reference SQL di payload menggunakan prefix `file:` relatif terhadap folder `payload/`:

```json
"query": "file:query/dashboard-sales/author-sales.sql"
```

Generator melakukan:
1. Strip prefix `file:` dan resolve path dari `payload/`
2. Baca isi SQL dan escape karakter berbahaya (backtick dan `${`)
3. Embed sebagai template literal di handler generated

---

## Output File Generated

Generator menghasilkan satu file handler Express di:

```
src/modules/{project}/{name}.js
```

File ini berisi satu route `POST /dashboard`. SQL setiap widget sudah di-embed sebagai string literal sehingga tidak ada disk read saat request masuk.

### Conflict Handling dengan `--force`

| `--force` | Behavior |
|-----------|----------|
| `false` (default) | Error stop; generator menyarankan `--force=true` |
| `true` | File lama di-archive ke `{name}.js.archive.{timestamp}`, lalu file baru ditulis |

---

## Dukungan Multi-Dialect

Dashboard mendukung tiga dialect database:

| Dialect | Status | Driver runtime |
|---------|--------|----------------|
| PostgreSQL | Default | `node-postgres` (`pg`) |
| MySQL | Didukung | `mysql2` |
| Oracle | Didukung | `oracledb` |
| SQLite | Ditolak | Valid untuk endpoint CRUD lain |

### Generate per Dialect

Pilih dialect via flag `--database` saat generate:

```bash
# PostgreSQL (default)
npx restforge dashboard create --project=my-project --name=dash-sales \
  --payload=dashboard-sales --database=postgres

# MySQL
npx restforge dashboard create --project=my-project --name=dash-sales \
  --payload=dashboard-sales --database=mysql

# Oracle
npx restforge dashboard create --project=my-project --name=dash-sales \
  --payload=dashboard-sales --database=oracle
```

### Perbedaan SQL per Dialect

SQL widget ditulis untuk satu dialect tujuan. Fungsi date dan pagination berbeda antar dialect:

| Operasi | PostgreSQL | MySQL | Oracle |
|---------|------------|-------|--------|
| Truncate ke bulan | `date_trunc('month', col)` | `DATE_FORMAT(col, '%Y-%m-01')` | `TRUNC(col, 'MM')` |
| Format tanggal | `TO_CHAR(col, 'YYYY-MM-DD')` | `DATE_FORMAT(col, '%Y-%m-%d')` | `TO_CHAR(col, 'YYYY-MM-DD')` |
| Pagination | `LIMIT N` | `LIMIT N` | `FETCH FIRST N ROWS ONLY` |
| Current date | `CURRENT_DATE` | `CURDATE()` | `SYSDATE` |
| Concatenation | `||` atau `CONCAT(...)` | `CONCAT(...)` | `||` atau `CONCAT(...)` |

### Bind Syntax Otomatis

Runtime mengonversi placeholder portable `:name` ke bind syntax dialect target secara otomatis. Pengguna cukup menulis `:name` di semua SQL widget.

| Dialect | Bind syntax | Contoh |
|---------|-------------|--------|
| PostgreSQL | Positional `$N` | `$1`, `$2`, `$3` |
| MySQL | Anonymous `?` | `?`, `?`, `?` |
| Oracle | Named `:N` | `:1`, `:2`, `:3` |

---

## Praktik Terbaik

### Penamaan Widget ID

Pakai snake_case dan deskripsikan semantik data, bukan visual:

| Direkomendasikan | Tidak Direkomendasikan |
|------------------|------------------------|
| `author_sales` | `bar_chart_1` |
| `monthly_revenue` | `area_chart_revenue` |
| `goal_progress` | `progress_bar_widget` |
| `sales_statistics` | `pie_chart_top_products` |

Widget id yang semantik mengisolasi backend dari pilihan visual frontend.

### Penamaan SQL File

Pakai kebab-case dan sesuaikan dengan widget id atau key:

```
query/dashboard-sales/
├── author-sales.sql                   ← widget "author_sales" → query
├── sales-statistics.sql               ← widget "sales_statistics" → query
├── sales-this-months-value.sql        ← widget "sales_this_months" → queries.value
└── sales-this-months-points.sql       ← widget "sales_this_months" → queries.points
```

### Performance SQL

Terapkan praktik berikut agar query widget efisien:

- Tambahkan `LIMIT N` untuk widget yang return array besar (`LIMIT 12` untuk bar/pie chart, `LIMIT 30` untuk timeseries).
- Buat index pada kolom filter (misalnya `outbound_date`, `status`).
- Pakai `COALESCE(SUM(x), 0)` agar scalar value tidak null.
- Gunakan `date_trunc` (PostgreSQL) atau ekuivalennya per dialect untuk grouping timeseries yang stabil.
- Pakai `generate_series` (PostgreSQL) atau recursive CTE (MySQL/Oracle) untuk menjamin jumlah baris timeseries konsisten meskipun ada periode tanpa transaksi.

### Tuning TTL Cache

| Skenario | TTL yang cocok | Alasan |
|----------|----------------|--------|
| Data bisa berubah dari sumber luar (batch job, ERP) | 60–300 detik | Cascade invalidation tidak cover perubahan dari luar |
| Dashboard internal dengan WRITE terkontrol | 300–1800 detik | Cascade invalidation menjamin freshness |
| Monthly report yang data sumbernya stabil | 1800–3600 detik | TTL panjang aman karena data jarang berubah |

### Strategi SQL per Dialect

Dua strategi penyimpanan SQL untuk project multi-dialect:

**Strategi A: Satu folder per dialect (direkomendasikan untuk widget kompleks)**

```
payload/
├── dashboard-sales-postgres.json
├── dashboard-sales-mysql.json
└── query/
    ├── postgres/
    │   └── author-sales.sql
    └── mysql/
        └── author-sales.sql
```

Regenerate dengan flag `--database` dan `--payload` yang sesuai per dialect target.

**Strategi B: SQL portable subset (untuk widget sederhana)**

Tulis SQL menggunakan subset yang valid di tiga dialect: `SUM`, `COUNT`, `GROUP BY`, `JOIN` tanpa fungsi date. Pertahankan satu file SQL dan ganti `--database` saat regenerate.

### Mengupdate SQL Widget

Saat SQL widget berubah:

1. Edit file SQL di `payload/query/dashboard-{name}/`.
2. Jalankan ulang generate dengan `--force=true`.
3. Restart server (atau pakai `--watch` mode untuk hot reload).

```bash
npx restforge dashboard create \
  --project=mini-inventory \
  --name=dash-sales \
  --payload=dashboard-sales \
  --force=true
```

Tidak perlu memodifikasi `{name}.js` secara manual. SQL sudah di-embed di code generated dan akan diperbarui setiap kali generate dijalankan ulang.

---

## Konvensi Tipe Response Cross-Dialect

Backend database mengembalikan tipe data yang berbeda per dialect:

- PostgreSQL `node-postgres` mengembalikan kolom `NUMERIC` sebagai string.
- MySQL `mysql2` mengembalikan `DECIMAL` sebagai string.
- Oracle `oracledb` mengembalikan `NUMBER` sebagai number JS native dengan key UPPERCASE.

Runtime menormalisasi response agar identik di tiga dialect:

| Konvensi | Behavior |
|----------|----------|
| Numeric → string | Helper `stringifyNumericDeep` mengubah setiap `typeof === 'number'` menjadi string |
| Key → lowercase | Helper `lowercaseKeysDeep` menormalisasi semua key object ke lowercase |

Contoh sebelum dan sesudah normalisasi (Oracle):

```
// Sebelum (Oracle driver)
{ "VALUE": 69700 }

// Sesudah (post-normalisasi, semua dialect)
{ "value": "69700" }
```

Frontend wajib `parseFloat()` atau `Number()` sebelum operasi aritmatika atau sebelum feed ke library chart yang mengharapkan tipe number.

---

## Keterbatasan

| Keterbatasan | Catatan |
|--------------|---------|
| SQLite tidak didukung | Hanya PostgreSQL, MySQL, dan Oracle; endpoint CRUD lain tetap mendukung SQLite |
| Inline SQL di payload | Ditolak validator; hanya file reference `file:...` yang diterima |
| Async parameter resolution | Parameter berasal dari request body; tidak ada pull otomatis dari header, cookie, atau session |
| Row-level security native | Handler tidak membaca `req._requestScope`; filter berbasis user identity dilakukan di middleware atau via SQL parameter eksplisit |
| Kontrol akses per widget | Tidak ada policy akses native per widget; dikontrol di level endpoint oleh auth middleware |

---

## Troubleshooting

### Endpoint Return HTTP 403

**Gejala:** `POST /api/{project}/{name}/dashboard` mengembalikan `{"success": false, "error": "Forbidden"}`

File `{name}.js` gagal load saat startup sehingga route tidak terdaftar dan request jatuh ke catch-all 403 handler.

Langkah verifikasi:
1. Cek log startup untuk error `MODULE_NOT_FOUND` atau syntax error.
2. Pastikan file `{name}.js` ada di `src/modules/{project}/`.
3. Pastikan generator memakai pattern `require('restforgejs/src/utils/...')`, bukan relative path.

---

### Widget Return `{ "error": "..." }` tapi Widget Lain OK

**Gejala:** Salah satu widget berisi `{ "error": "..." }`, widget lain berjalan normal.

Ini adalah behavior yang diharapkan. SQL widget tersebut throw saat eksekusi; top-level `success: true` tetap karena partial degradation lebih baik daripada total failure. Penyebab umum:

- Tabel atau kolom tidak ada di database
- Syntax error di SQL
- Permission database tidak cukup
- Placeholder `:param` tidak resolve

Langkah verifikasi:
1. Lihat pesan `error` untuk hint root cause.
2. Cek log database untuk query yang dieksekusi.
3. Jalankan SQL widget secara manual via psql/mysql client untuk mereproduksi error.

---

### Validator Menolak Field `widgetType`

**Gejala:** `Field 'widgetType' is not allowed in backend payload (frontend concern)`

Solusi: pindahkan `widgetType`, `layout`, `title`, `subtitle`, `color` ke konfigurasi frontend. Payload backend hanya boleh memuat `widgets`, `params`, dan `cache`.

---

### Scalar Tidak Collapse

**Gejala:** Diharapkan `"value": "100"`, mendapat `"value": { "value": "100" }`.

SQL menghasilkan multi-column meskipun hanya 1 row sehingga collapse rule menghasilkan object.

```sql
-- Salah: multi-column → object
SELECT SUM(amount) AS value, COUNT(*) AS count FROM sales;

-- Benar: single column → scalar primitive
SELECT SUM(amount) AS value FROM sales;
```

---

## Metadata

Setiap kali generate dijalankan, runtime mencatat dashboard ke `metadata/{project}.json` dengan `type: "dashboard"`. Entry ini dipakai oleh cache invalidation registry.

| Field metadata | Keterangan |
|----------------|------------|
| `type` | Selalu `"dashboard"` |
| `widgetCount` | Jumlah widget di payload |
| `widgetIds` | Array id widget |
| `paramsContract` | Summary params (`keyCount`, `keys`) |
| `databaseType` | Dialect yang digunakan |
| `statistics.timesGenerated` | Counter berapa kali generate dijalankan |

Field metadata dashboard berbeda dari endpoint CRUD:

| Field | CRUD | Dashboard |
|-------|------|-----------|
| `type` | `module` | `dashboard` |
| `tableName` | ada | tidak ada |
| `actions` | ada | tidak ada |
| `widgetCount`, `widgetIds` | tidak ada | ada |
| `paramsContract` | tidak ada | ada |
