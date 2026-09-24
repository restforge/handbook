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
    "autoRefresh": { "interval": 30 },
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
| `autoRefresh` | object | — | Polling reload otomatis tiap `interval` detik (`{ "interval": <detik> }`); tidak dapat digabung dengan `enableLiveSync` pada page yang sama; tidak didukung plugin `vanilla-js-basic` |
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
| `width` | string | | Lebar CSS kontrol filter, mis. `"220px"` atau `"100%"`. Opsional. Lihat [Lebar Filter (`width`)](#lebar-filter-width) untuk default dan perbedaan antar plugin |

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
| `width` | string | | Lebar CSS kontrol filter, mis. `"220px"` atau `"100%"`. Opsional. Lihat [Lebar Filter (`width`)](#lebar-filter-width) untuk default dan perbedaan antar plugin |
| `dependsOn` | string | | Nama filter induk (nilai `name` filter lain di `dataFilters[]` yang sama). Bila diisi, filter ini menjadi cascade — lihat [Filter Cascade (`dependsOn`)](#filter-cascade-dependson) |
| `display` | string | | Posisi filter: `"inline"` (di toolbar) atau `"dropdown"` (di popup "More Filters"). Opsional. Tanpa properti ini, posisi ditentukan positional (maks 2 slot inline gabungan dengan `statusFilter`, sisanya ke popup). Override eksplisit mem-bypass kuota — lihat [Maks 2 Filter Inline + Filter Button](#maks-2-filter-inline--filter-button) |

Lihat [`data-source.md`](./data-source.md) untuk detail struktur `dataSource`.

### Lebar Filter (`width`)

Properti `width` opsional pada `statusFilter` maupun tiap entri `dataFilters[]` mengatur
lebar kontrol filter di toolbar. Nilainya string CSS width apa saja (`"220px"`, `"18rem"`,
`"100%"`). Berguna ketika opsi filter berteks panjang, mis. nama perusahaan, yang membungkus
pada lebar default.

```json
"dataFilters": [
    {
        "name": "company_id",
        "field": "company_id",
        "label": "Company",
        "width": "220px",
        "dataSource": { "type": "api", "endpoint": "/api/company", "valueField": "id", "textField": "name" }
    }
]
```

**Behavior `width` seragam antar plugin (mode inline).** Bila `width` diisi, nilainya
dipakai apa adanya untuk setiap tipe filter (select, status, date, dateRange, monthYear)
di semua plugin yang mendukung tipe tersebut. Untuk filter dua kontrol (dateRange dengan
input from/to, monthYear dengan select bulan/tahun), `width` diterapkan ke masing-masing
kontrol.

Default saat `width` tidak diisi mengikuti tema tiap plugin:

| Plugin | Default saat `width` kosong |
|--------|-----------------------------|
| `vanilla-js-auth` (Metronic) | Lebar inline per tipe: `125px` (select/status), `160px` (date), `140px` (dateRange, tiap input), `140px`+`100px` (monthYear: bulan+tahun) |
| `vanilla-js-custom` | select/date mengikuti class CSS `.filter-select`; dateRange `120px` (tiap input); monthYear `100px` (tiap select) |
| `vanilla-js-basic` | select mengikuti class CSS `.filter-select`. Tidak mendukung tipe date/dateRange/monthYear (semua filter dirender sebagai `select`) |

Catatan:

- **`vanilla-js-basic`** tidak mengenal `filterType`, jadi hanya filter select yang berfungsi
  penuh; `width` tetap dihormati untuk filter select-nya.
- **Mode `display: "dropdown"`**: ketiga plugin memakai lebar penuh (`100%`) untuk semua tipe
  filter, sehingga `width` tidak diterapkan di mode ini (berlaku seragam antar plugin).

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

### Filter Cascade (`dependsOn`)

Default-nya, setiap `dataFilters[]` berdiri sendiri — opsi dropdown dimuat penuh sekali di awal, tidak bergantung filter lain. Untuk filter berjenjang (mis. Provinsi → Kabupaten/Kota → Kecamatan), tambahkan `dependsOn` yang menunjuk ke `name` filter induk:

```json
"dataFilters": [
    {
        "name": "province_id",
        "field": "province_id",
        "label": "Provinsi",
        "dataSource": { "type": "api", "resource": "province", "select": ["province_id", "province_name"] }
    },
    {
        "name": "regency_id",
        "field": "regency_id",
        "label": "Kabupaten/Kota",
        "dependsOn": "province_id",
        "dataSource": { "type": "api", "resource": "regency", "select": ["regency_id", "regency_name"] }
    }
]
```

Behavior:

| Aspek | Behavior |
|-------|----------|
| Saat halaman dimuat | Filter dengan `dependsOn` TIDAK ikut eager-load; opsinya cuma "All" sampai induknya dipilih |
| Saat filter induk berubah | Opsi filter anak di-fetch ulang via endpoint `/lookup` dengan `where: [{"key": "<dependsOn>", "value": "<nilai induk>"}]` |
| Saat filter induk dikosongkan | Opsi filter anak kembali ke "All" saja (tidak fetch) |
| Cascade berjenjang | Mengubah filter di tengah rantai otomatis mengosongkan seluruh filter di bawahnya, tidak cuma anak langsung |
| Cakupan tipe | Hanya berlaku untuk filter select dengan `dataSource.type: "api"`. Diabaikan untuk `dataSource.type: "static"` atau filter date/dateRange/monthYear |
| Persistensi localStorage | Filter dengan `dependsOn` TIDAK disimpan/dipulihkan dari localStorage — menghindari race condition karena opsi anak belum tentu selesai dimuat saat restore mencoba mengisi value-nya. Filter induk tetap tersimpan/dipulihkan seperti biasa |

`dependsOn` wajib menunjuk ke `name` filter lain di array `dataFilters[]` yang sama. Bila menunjuk ke nama yang tidak ada, filter tersebut tidak pernah ter-populate (tidak menghasilkan error validasi, cukup diam).

Mekanisme cascade serupa juga tersedia untuk field `select` pada form (bukan hanya filter toolbar) — lihat [Select Cascade (`dependsOn`)](./field-types.md#select-cascade-dependson) di `field-types.md`.

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

### Active Filters Bar (Badge dan Notice Chip)

Setiap halaman yang menampilkan filter button (lihat [Maks 2 Filter Inline + Filter
Button](#maks-2-filter-inline--filter-button) di atas) otomatis mendapat dua elemen
tambahan, tanpa perlu konfigurasi apa pun di payload:

1. **Badge angka** pada tombol Filter/"More Filters" — menghitung jumlah filter yang
   sedang aktif (nilainya terisi, bukan default kosong), mencakup filter inline
   maupun yang ada di dalam filter button.
2. **Notice bar "Active filters"** di bawah toolbar — menampilkan satu chip per
   filter aktif berisi label dan nilai terpilih. Tiap chip bisa dihapus individual
   (ikon ×), plus link "Clear all" untuk mengosongkan semua filter sekaligus.

Behavior:

| Aspek | Behavior |
|-------|----------|
| Kapan muncul | Hanya saat ada minimal satu filter aktif; bar tersembunyi bila semua filter kosong |
| Hapus satu chip | Mengosongkan filter tersebut saja, lalu DataTable di-reload |
| Clear all | Mengosongkan SEMUA filter (inline, dropdown, dan status sekaligus), lalu DataTable di-reload |
| `statusFilter` | Ikut dihitung badge dan tampil sebagai chip bila ada nilai terpilih |
| Cakupan plugin | Muncul di ketiga plugin (`vanilla-js-auth`, `vanilla-js-custom`, `vanilla-js-basic`), mengikuti idiom visual masing-masing (Metronic vs CSS custom) |

Fitur ini tidak bisa dimatikan lewat payload — otomatis aktif kapan pun filter
button muncul (lihat tabel skenario A/B/C di atas).

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

Halaman dengan `enableLiveSync` aktif juga mendapat tombol Refresh di toolbar beserta penanda waktu `Updated HH:MM · live`, mengikuti aturan yang sama dengan `autoRefresh` (lihat [`autoRefresh`](#autorefresh) di bawah).

Lihat [`live-sync.md`](./live-sync.md) untuk konfigurasi blok root `liveSync`.

## `autoRefresh`

Polling reload otomatis untuk halaman yang datanya bisa berubah oleh pihak lain, tetapi aplikasi tidak punya server WebSocket untuk Live Sync, misalnya halaman antrean job atau log yang diproses oleh proses lain.

```json
"features": {
    "autoRefresh": { "interval": 30 }
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `interval` | integer | ✓ | Jeda polling dalam detik, minimal 10 |

Behavior:

- Toolbar menampilkan tombol Refresh beserta penanda waktu `Updated HH:MM · auto every Ns`
- Polling ditunda selama panel/modal form tambah atau ubah terbuka, dan saat tab browser tersembunyi
- Tombol nonaktif (disabled) selama request reload sedang berjalan

Didukung oleh plugin `vanilla-js-auth` dan `vanilla-js-custom`. Plugin `vanilla-js-basic` mengabaikan `autoRefresh`; generator menampilkan warning.

`autoRefresh` tidak boleh digabung dengan `enableLiveSync` pada page yang sama, pilih salah satu mekanisme. Validator menghasilkan pesan berikut:

```
[<pageId>] features.autoRefresh must be an object with an 'interval' field (seconds)
[<pageId>] features.autoRefresh.interval must be an integer (seconds)
[<pageId>] features.autoRefresh.interval must be at least 10 seconds
[<pageId>] features.autoRefresh cannot be combined with features.enableLiveSync on the same page; choose one refresh mechanism
[<pageId>] features.autoRefresh is not supported by plugin 'vanilla-js-basic' and will be ignored
```

Empat baris pertama adalah error yang menggagalkan validasi. Baris terakhir adalah warning: payload tetap valid, tetapi `autoRefresh` diabaikan oleh generator untuk plugin `vanilla-js-basic`.

## Tombol Refresh dan Reload Otomatis

Tombol Refresh hanya dirender bila halaman punya `enableLiveSync` atau `autoRefresh` aktif. Halaman tanpa keduanya tidak punya tombol Refresh, tetapi data tetap dimuat ulang otomatis setelah simpan, ubah, hapus, dan saat tab browser kembali aktif setelah tersembunyi lebih dari 5 menit.

Aturan ini berlaku untuk seluruh halaman CRUD hasil generate: tombol Refresh adalah pelengkap mekanisme reload otomatis, bukan fitur yang bisa berdiri sendiri tanpa mekanisme itu.

## Perilaku Form Tambah dan Ubah

Satu alur tambah data memakai kalimat yang sama di empat titik: tombol tambah di baris judul halaman, judul panel form, tombol penyimpan di footer panel, dan tombol pada daftar kosong. Kalimatnya `{addVerb} {pageSubject}` (default `Add {pageTitle}`), form ubah memakai judul `Edit {pageSubject}` dengan tombol penyimpan `Save Changes`, dan mode lihat memakai `View {pageSubject}`. Kedua properti itu dijelaskan di [`page-anatomy.md`](./page-anatomy.md#label-alur-tambah-data).

### Keluar dari Form

| Kondisi | Perilaku |
|---------|----------|
| Form belum diubah | Tombol `Cancel`, ikon silang, dan tombol Escape menutup panel langsung |
| Form sudah diubah | Ketiga pintu keluar itu menampilkan dialog `Discard changes?`. Pilihan `Keep Editing` mendapat fokus awal dan kembali ke form, `Discard` menutup panel dan membuang isian |
| Mode lihat | Tombol keluarnya berlabel `Close` dan tidak pernah memunculkan dialog |

Nilai yang diisi sistem tidak dihitung sebagai perubahan pengguna, yaitu `defaultValue`, nomor hasil ID generation, dan data yang dimuat saat membuka mode ubah atau duplikat. Dialog baru muncul setelah pengguna sendiri mengubah isi field.

### Selama Simpan dan Saat Gagal

| Aspek | Perilaku |
|-------|----------|
| Tombol penyimpan | Nonaktif dengan indikator berputar selama request berjalan, labelnya tidak berubah |
| Tombol `Cancel` dan ikon silang | Tetap dapat diklik. Halaman tidak ditutup overlay |
| Klik kedua pada tombol penyimpan | Diabaikan selama request pertama belum selesai |
| Respons 400 atau 409 yang membawa `details` | Pesan per field tampil di bawah field terkait, ringkasan tampil di atas footer panel, dan fokus berpindah ke field bermasalah pertama. Panel tetap terbuka dan isian dipertahankan |
| Pesan untuk field yang tidak punya input di form | Ikut tampil sebagai butir di ringkasan |
| Kegagalan jaringan | Ringkasan di panel ditambah pesan singkat |
| Simpan berhasil | Panel tertutup, pesan singkat berbunyi `{pageSubject} added` (`created`, `uploaded`, atau `invited` mengikuti verba) atau `{pageSubject} updated` |

Bentuk `details` pada respons 400 dan 409 dijelaskan di [`../rdf/field-validation.md`](../rdf/field-validation.md).

### Simpan Perubahan pada Form Ubah

Tombol `Save Changes` pada form ubah membandingkan isian saat ini dengan nilai yang dimuat saat form dibuka, termasuk baris detail pada halaman master-detail. Tanpa satu pun perubahan, klik tombol itu langsung menutup panel dan menampilkan pesan singkat `No changes to save`, tanpa mengirim request sama sekali. Respons server `noChanges: true` (RDF ber-`concurrency`, lihat [`../rdf/concurrency.md`](../rdf/concurrency.md)) diperlakukan sama: panel tertutup dengan pesan yang sama, tanpa memuat ulang list dan tanpa menyorot baris.

Halaman yang mendeklarasikan [`versionField`](./page-anatomy.md#koneksi-ke-backend) mengirim token versi setiap `Save Changes` dan menolak menimpa data saat server membalas konflik: panel tetap terbuka dengan isian utuh, ringkasan di atas footer panel berganti gaya peringatan berisi `This record was modified by <pengubah> at <waktu> after you opened it.` (bagian pengubah atau waktu yang tidak diketahui server dihilangkan dari kalimat), dan tombol `Reload latest values` menggantikan isian dengan data terbaru serta memperbarui token versi. Tidak ada penimpaan maupun penggabungan otomatis; pengguna sendiri yang memutuskan menyimpan ulang setelah meninjau data terbaru. Halaman tanpa `versionField` tidak pernah menampilkan konflik ini, sama seperti sebelum fitur ini ada.

### Daftar Kosong

Tabel yang belum punya satu baris pun menampilkan kalimat `No {pageSubject} yet` dengan nama objek berhuruf kecil, beserta tombol tambah berlabel sama dengan tombol di baris judul. Tombol itu mengikuti gating permission yang sama dengan tombol header: pada plugin ber-auth ia muncul hanya untuk pengguna yang berhak membuat data. Tabel yang kosong karena pencarian atau filter menampilkan `No matching {pageSubject} found` tanpa tombol, karena yang dibutuhkan pengguna di situ adalah melonggarkan saringan, bukan menambah data.

## Perilaku List Setelah Simpan

### Urutan Baris dan Tie-breaker

Setiap request `/datatables` mengirim kolom urut pilihan pengguna, lalu primary key menaik sebagai penentu urutan terakhir. Dua baris yang nilai kolom urutnya sama karena itu selalu tampil dalam urutan yang sama antar request, dan hitungan posisi baris baru di bawah ini tidak meleset satu baris.

### Setelah Data Baru Tersimpan

List tidak sekadar dimuat ulang. Halaman tempat baris baru berada dihitung di bawah pencarian, filter, dan urutan yang sedang aktif, list berpindah ke halaman itu, lalu barisnya disorot beberapa detik dan digulir ke tengah layar.

| Kondisi | Yang terjadi |
|---------|--------------|
| Baris baru masuk saringan yang sedang aktif | List berpindah ke halamannya, baris disorot dan digulir ke tengah |
| Baris baru tersaring keluar oleh pencarian atau filter | Pesan sukses membawa tautan `Show in list`. Klik pada tautan itu mengosongkan pencarian dan filter lalu menampilkan barisnya. Selama tautan belum diklik, saringan tidak dicabut |
| Pencarian posisi gagal | List dimuat ulang seperti sebelumnya dan pesan sukses tetap tampil. Kegagalan ini tidak pernah dilaporkan sebagai kegagalan simpan |

Lompat ke baris baru dijamin hanya bila kolom urut yang sedang aktif berasal dari tabel halaman itu sendiri dan nilainya tidak kosong, karena nilai pembandingnya dibaca dari record yang dikembalikan `/create`. Kolom urut hasil JOIN dan kolom bernilai kosong membuat list jatuh ke pemuatan ulang biasa, tanpa error dan tanpa pesan tambahan.

### Setelah Ubah Berhasil

List di halaman yang sama dimuat ulang, lalu baris yang baru saja diubah disorot di tempatnya beberapa detik dengan warna sorot yang sama dengan baris baru (lihat [Badge `New`](#badge-new) di bawah untuk pola warna serupa) dan digulir ke tengah layar, disertai pesan singkat `{pageSubject} updated`. Berbeda dari alur data baru, baris ubah tidak pernah pindah halaman: posisinya sudah diketahui dari primary key hasil respons [`/update`](../../api-spec/endpoint-update.md) (atau [`/update-composite`](../../api-spec/endpoint-update-composite.md) untuk master-detail), bukan dari pencarian ulang seperti baris baru.

Bila kolom urut yang sedang aktif berubah nilainya akibat perubahan itu sendiri, baris bisa saja berpindah posisi setelah list dimuat ulang sehingga sorotannya jatuh di tempat baru; ini batas mode server-side (list selalu dimuat ulang dari server, tidak pernah disusun ulang di browser) dan bukan kegagalan.

### Ubah, Hapus, dan Jumlah Baris

| Aksi | Yang terjadi pada list |
|------|------------------------|
| Ubah | Halaman yang sedang tampil dimuat ulang; pencarian, filter, urutan, dan nomor halaman tetap. Detail sorotan baris ada di atas |
| Hapus | Sama dengan ubah |
| Hapus baris terakhir pada halaman terakhir | List berpindah ke halaman sebelumnya, bukan menampilkan halaman kosong |

Jumlah baris selalu tampil di ketiga plugin dengan format `1–10 of 128`, dan `0 of 0` saat tabel kosong.

### Badge `New`

Baris yang dibuat hari ini menurut tanggal server mendapat badge `New` di sebelah kolom identitas, dengan tooltip `Created today by <pembuat>` yang mengambil nama dari `created_by`. Tanpa `created_by` di data baris, tooltip berbunyi `Created today`. Badge hanya ikut pada nilai tampilan: nilai untuk pengurutan, filter, dan export tidak membawa markup badge. Badge berhenti muncul dengan sendirinya keesokan harinya karena kondisinya tidak lagi terpenuhi, tanpa penyimpanan apa pun di sisi client.

Dua prasyarat berada di luar UDF:

| Prasyarat | Keterangan |
|-----------|------------|
| `created_at` ikut di data baris | Kolom audit ini harus ikut keluar di `datatablesQuery` payload backend meski tidak ditampilkan sebagai kolom tabel. Lihat [`../rdf/audit-columns.md`](../rdf/audit-columns.md) |
| Response `/datatables` membawa `timestamp` | Dipakai sebagai acuan tanggal hari ini menurut server, bukan jam browser. Lihat [`../../api-spec/endpoint-datatables.md`](../../api-spec/endpoint-datatables.md) |

Bila salah satunya tidak ada, badge tidak dirender dan sisa halaman berjalan seperti biasa. Badge dimatikan per halaman lewat `showNewBadge: false`.

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
4. Tombol Refresh (hanya bila `enableLiveSync` atau `autoRefresh` aktif)

Tombol tambah berlabel `{addVerb} {pageSubject}` (default `Add {pageTitle}`) tidak termasuk isi toolbar. Tempatnya di baris judul halaman, sejajar judul, seperti dijelaskan di [Perilaku Form Tambah dan Ubah](#perilaku-form-tambah-dan-ubah).

---

← [`field-rows.md`](./field-rows.md) | [Selanjutnya: `master-detail.md`](./master-detail.md) →
