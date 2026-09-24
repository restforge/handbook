# Read — Endpoint Pengambilan Data

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/read` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL, MySQL, Oracle |
| **Mode Operasi** | Paginasi (dengan `page`) dan Non-Paginasi (tanpa `page`) |
| **Batas Non-Paginasi** | `limit` maksimum 5000 (default), dapat diatur lewat `READ_MAX_LIMIT`; mode tanpa batas opsional (lihat [Batas Jumlah Baris Mode Non-Paginasi](#batas-jumlah-baris-mode-non-paginasi)) |
| **Parameter Utama** | `page`, `per_page`, `limit`, `select`, `search_value`, `search_by`, `sort_columns`, `where` |
| **Cache** | Redis (key: `rf:{project}:{endpoint}:list:{hash}`) |
| **Default Scope** | Diterapkan jika dikonfigurasi di payload |

---

## Ikhtisar

Endpoint `/read` adalah endpoint universal untuk mengambil data dengan dukungan **dua mode operasi**: paginasi dan non-paginasi. Mode ditentukan secara otomatis berdasarkan keberadaan parameter `page` di request body. Jika `page` dikirim maka mode paginasi aktif, jika tidak maka mode non-paginasi yang digunakan.

Endpoint ini mendukung pencarian teks, pengurutan multi-kolom, kondisi WHERE kompleks, dan pemilihan kolom selektif (`select`). Sumber data ditentukan berdasarkan prioritas resolusi: `viewName` → `viewQuery` → `tableName`.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/supplier/read
POST http://localhost:3000/api/mini-inventory/item-product/read
```

---

## Mode Operasi

| Mode | Kondisi | Keterangan |
|------|---------|------------|
| **Paginasi** | `page` dikirim di request body | Data per halaman, response memiliki blok `pagination` |
| **Non-Paginasi** | `page` **tidak** dikirim | Semua data yang cocok dikembalikan sampai batas server (default 5000, lihat [Batas Jumlah Baris Mode Non-Paginasi](#batas-jumlah-baris-mode-non-paginasi)), tanpa blok `pagination`. Respons menandai bila hasil terpotong |

Tidak perlu parameter tambahan untuk memilih mode. Cukup sertakan `page` untuk mode paginasi, atau hilangkan untuk mode non-paginasi.

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Default | Batasan | Keterangan |
|-----------|------|:-----:|---------|---------|------------|
| `page` | number | Tidak | — | >= 1 | Nomor halaman. Jika dikirim → mode paginasi |
| `per_page` | number | Tidak | `10` | 1 — 100 | Jumlah data per halaman. Hanya berlaku jika `page` dikirim |
| `limit` | number \| `"all"` | Tidak | `1000` | 1 — `READ_MAX_LIMIT` (default 5000) | Batas record di mode non-paginasi. Nilai di atas batas dipotong ke batas dan ditandai `truncated: true`; nilai kurang dari 1, `0`, atau bukan angka jatuh ke `1000`. Literal `"all"` meminta seluruh baris dan hanya berlaku bila server mengizinkan (lihat [Batas Jumlah Baris Mode Non-Paginasi](#batas-jumlah-baris-mode-non-paginasi)); bila tidak diizinkan, dipotong ke batas tanpa error. Diabaikan jika `page` dikirim |
| `select` | array | Tidak | Semua kolom | Elemen harus ada di `readableFields`. Tidak perlu memuat kolom pengurut | Kolom spesifik yang ditampilkan |
| `search_value` | string | Tidak | `""` | Maks 255 karakter | Kata kunci pencarian teks |
| `search_by` | string | Tidak | Kolom pertama payload | Harus ada di `readableFields` | Kolom target pencarian |
| `sort_columns` | array | Tidak | Primary key ASC | Lihat [Format Sort Columns](README.md#format-sort-columns) | Pengurutan data |
| `where` | array / object | Tidak | `null` | Mendukung grup bersarang, lihat [Format WHERE](README.md#format-where-clause) | Kondisi filter data |

### Validasi Parameter

| Parameter | Aturan | Perilaku jika Tidak Valid |
|-----------|--------|--------------------------|
| `page` | Harus >= 1 | Error 400: `"Page must be greater than 0"` |
| `per_page` | Harus 1 — 100 | Error 400: `"Per page must be between 1 and 100"` |
| `limit` | Dipotong ke batas `READ_MAX_LIMIT` (default 5000); literal `"all"` tanpa izin server juga dipotong ke batas | Tidak ada error 400, nilai otomatis disesuaikan |
| `select` | Setiap elemen dicek terhadap `readableFields` | Error 400: `"Invalid select fields"` |
| `search_value` | Maks 255 karakter | Error 400: `"Search value must not exceed 255 characters"` |
| `search_by` | Harus ada di `readableFields` | Error 400: `"Invalid search field"` |
| `sort_columns` | Kolom dicek terhadap `readableFields` | Kolom tidak valid diabaikan (silent skip) |
| `where` (key) | Key harus ada di `readableFields` | Error 400: `"Invalid field: <nama_kolom>"` — fail-closed, tidak di-filter diam-diam |

### Contoh Request

**1. Paginasi sederhana:**
```json
{
  "page": 1,
  "per_page": 20
}
```

**2. Paginasi dengan pencarian dan pengurutan:**
```json
{
  "page": 1,
  "per_page": 10,
  "search_value": "maju",
  "search_by": "supplier_name",
  "sort_columns": [
    { "column": "supplier_name", "direction": "ASC" }
  ]
}
```

**3. Non-paginasi dengan filter WHERE:**
```json
{
  "limit": 500,
  "where": [
    { "key": "is_active", "value": "true" }
  ],
  "sort_columns": [
    { "column": "created_date", "direction": "DESC" }
  ]
}
```

**4. Dengan kolom selektif:**
```json
{
  "page": 1,
  "per_page": 50,
  "select": ["supplier_id", "supplier_code", "supplier_name"]
}
```

**5. Dengan WHERE kompleks:**
```json
{
  "page": 1,
  "per_page": 25,
  "where": {
    "logic": "AND",
    "conditions": [
      { "key": "selling_price", "operator": ">=", "value": 10000 },
      { "key": "category_id", "operator": "IN", "value": ["cat-1", "cat-2"] },
      { "key": "product_name", "operator": "LIKE", "value": "baut", "sensitive": false }
    ]
  }
}
```

---

## Format Response

### Response Sukses — Mode Paginasi

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": [
    { "supplier_id": "...", "supplier_code": "SUP-001", "supplier_name": "PT Maju Jaya", "is_active": true },
    { "supplier_id": "...", "supplier_code": "SUP-002", "supplier_name": "CV Berkah", "is_active": true }
  ],
  "count": 2,
  "pagination": {
    "current_page": 1,
    "per_page": 10,
    "total_records": 42,
    "total_pages": 5,
    "has_next": true,
    "has_previous": false
  }
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika query berhasil |
| `data` | array | Array berisi data hasil query |
| `count` | number | Jumlah record di halaman ini (`data.length`) |
| `pagination.current_page` | number | Halaman yang sedang ditampilkan |
| `pagination.per_page` | number | Jumlah data per halaman |
| `pagination.total_records` | number | Total record setelah filter diterapkan |
| `pagination.total_pages` | number | Total halaman tersedia |
| `pagination.has_next` | boolean | `true` jika ada halaman berikutnya |
| `pagination.has_previous` | boolean | `true` jika ada halaman sebelumnya |

### Response Sukses — Mode Non-Paginasi

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": [
    { "supplier_id": "...", "supplier_code": "SUP-001", "supplier_name": "PT Maju Jaya" },
    { "supplier_id": "...", "supplier_code": "SUP-002", "supplier_name": "CV Berkah" }
  ],
  "count": 2,
  "total": 2,
  "limit": 1000,
  "maxLimit": 5000,
  "truncated": false
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika query berhasil |
| `data` | array | Array berisi data hasil query |
| `count` | number | Jumlah record yang dikembalikan |
| `total` | number | Jumlah seluruh baris yang cocok dengan filter, terlepas dari `limit` |
| `limit` | number \| null | Batas efektif yang dipakai server untuk permintaan ini, yaitu nilai `limit` permintaan setelah dipotong ke batas. `null` pada mode tanpa batas |
| `maxLimit` | number \| null | Batas yang berlaku di server (`READ_MAX_LIMIT`, default 5000). `null` bila server dikonfigurasi ke mode tanpa batas |
| `truncated` | boolean | `true` bila `count` lebih kecil dari `total`, yaitu ada baris yang cocok tetapi tidak ikut dikirim |

> **Catatan:** Blok `pagination` **tidak muncul** di mode non-paginasi. Sebaliknya, field `total`, `limit`, `maxLimit`, dan `truncated` hanya ada di mode non-paginasi; mode paginasi tetap memakai blok `pagination`.

### Response Error

#### 400 — Nilai Tanggal-Waktu Tidak Valid

Terjadi ketika nilai field `date`, `timestamp`, atau `timestamptz` pada filter `where` tidak cocok dengan pola aktif (`DATEFORMAT`/`DATETIMEFORMAT`) maupun bentuk ISO. Tanggal saja pada field `timestamp`/`timestamptz`, tanggal kalender yang tidak nyata, jam di luar rentang, dan jam lokal pada celah atau tumpang-tindih DST termasuk kasus ini.

```json
{
  "success": false,
  "error": "Invalid datetime",
  "code": "INVALID_DATETIME",
  "message": "Invalid datetime value '18/09/2026': type 'timestamp' requires a time component; a date-only value is not accepted. Expected an ISO 8601 value or format 'dd/MM/yyyy HH:mm:ss'",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

Aturan lengkap ada di [`catalogs/rdf/datetime-fields.md`](../catalogs/rdf/datetime-fields.md#format-error).

#### 400 — Parameter Tidak Valid

Validasi parameter paginasi dan pencarian (`page`, `per_page`, `search_value`, `search_by`) menghasilkan format khusus **tanpa** field `error` dan **tanpa** `timestamp`. Key di dalam object `errors` adalah nama parameter terkait:

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": {
    "page": ["Page must be greater than 0"]
  }
}
```

Contoh isi `errors` lainnya (key menyesuaikan parameter yang gagal):
- `"per_page": ["Per page must be between 1 and 100"]`
- `"search_value": ["Search value must not exceed 255 characters"]`
- `"search_by": ["Invalid search field. Valid fields: supplier_id, supplier_code, ..."]`

#### 400 — WHERE / Sort Tidak Valid

Error yang muncul saat membangun query (mis. format WHERE atau seluruh kolom sort tidak valid) dikembalikan dengan format berikut:

```json
{
  "success": false,
  "error": "Bad Request",
  "message": "Invalid where conditions: ...",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

#### 400 — Field Select Tidak Valid

```json
{
  "success": false,
  "error": "Invalid select fields",
  "message": "Invalid field(s): field_tidak_ada",
  "validFields": ["supplier_id", "supplier_code", "supplier_name", "is_active"],
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

> Field `validFields` hanya disertakan saat `NODE_ENV=development`. Di produksi, field ini tidak ditampilkan (anti-disclosure). Berlaku untuk semua respons penolakan whitelist (select, where, search, sort).

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while fetching supplier list data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-03-30T10:30:00.000Z"
}
```

---

## Batas Jumlah Baris Mode Non-Paginasi

### Batas Bawaan dan Konfigurasi

Server membatasi jumlah baris maksimum yang dikembalikan dalam satu permintaan non-paginasi, terlepas dari nilai `limit` yang dikirim client. Batas defaultnya 5000 dan dapat diatur lewat variabel environment `READ_MAX_LIMIT` di file config project (mis. `config/db-connection.env`). Nilai environment yang tidak valid (bukan angka, `0`, atau negatif) jatuh ke default 5000. Default `limit` permintaan (`1000`) tidak berubah oleh pengaturan ini.

Bila `limit` yang diminta melebihi batas, server tidak menolak permintaan dengan error. Permintaan tetap diproses dengan jumlah baris dipotong ke batas, dan response menandainya lewat `truncated: true` beserta `limit` efektif yang dipakai dan `maxLimit` yang berlaku. Contoh permintaan `limit: 9000` terhadap data dengan `total: 6491` (batas server default 5000):

```json
{
  "success": true,
  "data": [ /* 5000 baris */ ],
  "count": 5000,
  "total": 6491,
  "limit": 5000,
  "maxLimit": 5000,
  "truncated": true
}
```

Setelah batas server dinaikkan (mis. `READ_MAX_LIMIT=8000`), permintaan `limit: 9000` yang sama terhadap data `total: 6491` tidak lagi terpotong:

```json
{
  "success": true,
  "data": [ /* 6491 baris */ ],
  "count": 6491,
  "total": 6491,
  "limit": 8000,
  "maxLimit": 8000,
  "truncated": false
}
```

### Mode Tanpa Batas

Selain menaikkan batas, tersedia jalur resmi untuk menarik seluruh baris yang cocok tanpa batas apa pun. Mode ini hanya aktif bila **dua syarat terpenuhi sekaligus**, supaya tidak menyala tanpa sengaja:

| Syarat | Cara |
|--------|------|
| Server mengizinkan | `READ_MAX_LIMIT=unlimited` (literal, bukan angka) di config project |
| Klien meminta secara eksplisit | `"limit": "all"` di body permintaan |

Perilaku kombinasinya:

| `READ_MAX_LIMIT` | `limit` permintaan | `limit` respons | `maxLimit` respons | `truncated` |
|-------------------|---------------------|------------------|----------------------|-------------|
| Angka atau tidak diisi | Angka | Dipotong ke batas | Batas yang berlaku | Sesuai keadaan |
| Angka atau tidak diisi | `"all"` | Dipotong ke batas | Batas yang berlaku | `true` bila masih ada baris tersisa |
| `unlimited` | Angka | Nilai permintaan itu sendiri (tidak dipotong) | `null` | Sesuai keadaan |
| `unlimited` | `"all"` | `null` | `null` | `false` |

Contoh respons mode tanpa batas aktif (`READ_MAX_LIMIT=unlimited` di server, `limit: "all"` di permintaan):

```json
{
  "success": true,
  "data": [ /* seluruh baris yang cocok */ ],
  "count": 6491,
  "total": 6491,
  "limit": null,
  "maxLimit": null,
  "truncated": false
}
```

> **Catatan:** Pada mode ini, query dijalankan tanpa klausa `LIMIT` dan hasilnya **tidak** disimpan ke cache list. Cache tetap berjalan seperti biasa untuk permintaan berbatas dan mode paginasi.

### Trade-off Mode Tanpa Batas

Mode tanpa batas tidak dijadikan perilaku bawaan karena beberapa hal berikut wajib dipertimbangkan sebelum diaktifkan:

| Trade-off | Penjelasan |
|-----------|------------|
| Memori server | Seluruh baris yang cocok ditampung sebagai satu array di memori proses sebelum diserialisasi menjadi satu respons JSON. Jumlah baris yang besar dengan banyak kolom dapat membebani memori secara signifikan, terlebih bila beberapa permintaan berjalan bersamaan |
| Ukuran respons dan timeout proxy | Respons dikirim setelah seluruh baris siap, bukan secara streaming. Reverse proxy, load balancer, atau klien dengan timeout pendek dapat memutus koneksi sebelum data selesai terkirim |
| Query COUNT tetap berjalan | Perhitungan `total` tetap dijalankan seperti mode berbatas; pada tabel besar tanpa index yang sesuai, ini bisa lebih lambat daripada query datanya sendiri |
| Tidak ada cursor untuk penarikan ulang | Bila koneksi terputus di tengah, klien harus mengulang permintaan dari awal; tidak tersedia mekanisme melanjutkan dari titik terakhir |
| Cache | Hasil mode tanpa batas tidak disimpan ke cache, sehingga tidak perlu menonaktifkan cache khusus untuk endpoint yang memakainya |
| Tanggung jawab bergeser ke klien | Karena server tidak lagi membatasi hasil, klien bertanggung jawab membatasi cakupan data lewat `where` (mis. bounding box wilayah) dan `select` (hanya kolom yang benar-benar dibutuhkan) |

### Pedoman Pemakaian

Mode tanpa batas cocok dipakai untuk kasus yang cakupan datanya sudah dibatasi lewat `where`, misalnya layer peta yang mengikuti bounding box viewport, ekspor data internal, atau sinkronisasi terjadwal antar-sistem. Mode ini **tidak** ditujukan untuk grid, dropdown, atau halaman listing pada umumnya.

Untuk kebutuhan agregasi pada zoom rendah (mis. peta yang menampilkan ringkasan per wilayah, bukan titik individual), pertimbangkan endpoint `/aggregate` sebagai gantinya. Alternatif lain yang lebih aman tanpa mengaktifkan mode tanpa batas adalah memanggil mode paginasi secara berulang dengan `per_page` besar, memakai `pagination.total_records` sebagai penghenti iterasi.

---

## Proyeksi Whitelist

Kolom yang dapat direferensikan di `select`, `search_by`, `sort_columns`, dan `where` dibatasi oleh **`readableFields`**: proyeksi whitelist yang diturunkan saat generate dari irisan `fieldName` dengan kolom output sumber-baca resolusi. Kolom fisik tabel yang tidak di-output sumber resolusi (mis. FK yang tidak masuk `viewQuery`) tidak dapat direferensikan. Bila soft-delete aktif, kolom soft-delete (`is_deleted`, `deleted_at`, `deleted_by`) otomatis ditambahkan ke `readableFields`.

Pada resource yang sumber bacanya berupa view atau `viewQuery`, `select` boleh tidak menyertakan primary key maupun kolom yang dipakai di `sort_columns`. Kolom pengurut ditangani otomatis saat query dijalankan lalu dibuang lagi dari setiap baris respons, sehingga kontrak "hanya kolom yang diminta yang dikembalikan" tetap berlaku persis. Pada rilis sebelumnya kombinasi `select` tanpa kolom pengurut pada sumber semacam itu membuat query gagal sehingga endpoint membalas 500.

## Sumber Data

Endpoint `/read` menentukan sumber query berdasarkan prioritas resolusi tiga tingkat:

| Prioritas | Properti Payload | Keterangan |
|:---------:|------------------|------------|
| 1 | `viewName` | Jika didefinisikan dan berbeda dari `tableName` → `SELECT * FROM viewName` |
| 2 | `viewQuery` | Digunakan jika `viewName` tidak ada dan berfungsi sebagai "virtual view" |
| 3 | `tableName` | Fallback terakhir → `SELECT * FROM tableName` |

> **Catatan:** `datatablesQuery` **tidak pernah** digunakan oleh endpoint `/read`. Query tersebut hanya untuk `/datatables`.

---

## Perilaku Cache

| Aspek | Nilai |
|-------|-------|
| Di-cache | Ya (jika Redis aktif dan `CACHE_ENABLED=true`) |
| Cache key | `rf:{project}:{endpoint}:list:{hash}` |
| Hash | MD5 (8 karakter) dari options request |
| TTL | Default 300 detik (configurable via `CACHE_TTL`) |
| Invalidasi | Otomatis setelah operasi `create`, `update`, atau `delete` |
| Mode tanpa batas | Selalu bypass, tidak dibaca maupun ditulis ke cache walau `CACHE_ENABLED=true` |

---

## Default Scope

Jika payload mengonfigurasi `defaultScope.read`, kondisi WHERE ditambahkan secara otomatis ke setiap query `/read`.

**Contoh konfigurasi payload:**
```json
{
  "defaultScope": {
    "read": { "is_active": true }
  }
}
```

**SQL yang dihasilkan:**
```sql
SELECT * FROM supplier WHERE is_active = true AND ... (user filters)
```

Default scope diterapkan **sebelum** kondisi WHERE dari request body, sehingga filter dari pengguna menjadi kondisi tambahan.

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Read (Deep Dive) | — | Dokumentasi lengkap endpoint read termasuk resolusi sumber data |
| Sort Column | — | Format dan perilaku pengurutan data |
| Cache | — | Konfigurasi dan perilaku cache Redis |
| Default Scope | — | Filter otomatis yang diterapkan pada setiap query |
| Query Declarative | — | Konfigurasi viewName, viewQuery, dan sumber data |
