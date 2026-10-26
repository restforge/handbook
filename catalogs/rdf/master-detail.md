# Field Master Detail (`masterDetail`)

Konfigurasi untuk operasi composite (header + detail dalam satu transaksi). Wajib ditetapkan jika action `createComposite`, `updateComposite`, atau `readComposite` aktif.

`masterDetail` punya dua bentuk. Bentuk satu detail menulis `detailTable` dan `detailConfig` langsung di level atas, seperti contoh di bawah. Bentuk `details[]` membawa beberapa detail sekaligus, dan dijelaskan di bagian [Beberapa Detail dalam Satu RDF](#beberapa-detail-dalam-satu-rdf-details).

```json
{
    "action": {
        "createComposite": true,
        "updateComposite": true,
        "readComposite": true
    },
    "masterDetail": {
        "enabled": true,
        "detailTable": "stock_inbound_item",
        "foreignKey": "stock_inbound_id",
        "cascadeDelete": true,
        "transactionMode": "required",
        "detailConfig": {
            "tableName": "stock_inbound_item",
            "primaryKey": "stock_inbound_item_id",
            "fieldName": [
                "stock_inbound_item_id", "stock_inbound_id",
                "line_number", "item_product_id", "qty_received",
                "unit_price", "total_amount"
            ],
            "detailQuery": "file:query/stock-inbound-detail.sql",
            "requiredFields": ["line_number", "item_product_id", "qty_received", "unit_price"],
            "autoCalculateFields": {
                "total_amount": {
                    "type": "calculated",
                    "formula": "qty_received * unit_price"
                }
            }
        },
        "headerCalculations": {
            "total_items":  { "type": "count", "source": "items.length" },
            "total_qty":    { "type": "sum",   "source": "items.qty_received" },
            "total_amount": { "type": "sum",   "source": "items.total_amount" }
        }
    }
}
```

Nama key `total_items`, `total_qty`, dan `total_amount` di contoh di atas hanya konvensi penamaan, bukan nama baku yang dikenali secara khusus. Key `headerCalculations` boleh berupa kolom header apa pun yang sudah ada di `fieldName` level root RDF; nilainya dihitung otomatis dari deklarasi `type` dan `source`, bukan ditebak dari nama key. Konfigurasi dengan kolom header `total` dan kolom detail `subtotal`, misalnya, menghasilkan agregat yang sama benarnya tanpa perlu mengganti nama kolom apa pun.

## Properti Top-Level `masterDetail`

Tabel berikut berlaku untuk bentuk satu detail. Pada bentuk `details[]`, hanya `enabled` dan `transactionMode` yang tetap di level atas, sedangkan properti lainnya berpindah ke setiap entri.

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `enabled` | boolean | Mengaktifkan fitur master-detail |
| `detailTable` | string | Nama tabel detail di database |
| `foreignKey` | string | Kolom FK di tabel detail yang merujuk ke header |
| `cascadeDelete` | boolean | Jika `true`, `delete` header lebih dulu menghapus seluruh detail items dalam transaksi yang sama. Berlaku di PostgreSQL, MySQL, SQLite, dan Oracle, dan tidak membutuhkan `ON DELETE CASCADE` pada foreign key detail |
| `transactionMode` | string | Hanya nilai `"required"` yang didukung. Semua operasi dalam satu transaction |
| `detailConfig` | object | Konfigurasi tabel detail (lihat di bawah) |
| `headerCalculations` | object | Field header yang dihitung secara otomatis dari detail items |

Blok `masterDetail` beserta `detailConfig` dapat digenerate otomatis dari introspeksi database lewat [`payload generate --detail=<table>`](../../commands/restforge-backend/payload/generate.md#master-detail-otomatis---detail). `headerCalculations` dan entry `autoCalculateFields` bertipe `calculated` tetap perlu dilengkapi manual setelahnya; lihat bagian [Pengisian Manual Setelah Generate](#pengisian-manual-setelah-generate---detail) di bawah.

## Beberapa Detail dalam Satu RDF (`details[]`)

Satu RDF composite dapat membawa beberapa detail, masing-masing dengan kardinalitas sendiri. Business Partner, misalnya, punya peran vendor dan peran customer yang paling banyak satu baris per partner, ditambah daftar kontak yang bisa berisi banyak baris. Untuk kebutuhan ini, `masterDetail` memakai array `details[]` sebagai pengganti pasangan `detailTable` dan `detailConfig` di level atas.

```json
{
    "tableName": "business_partner",
    "primaryKey": "business_partner_id",
    "fieldName": ["business_partner_id", "partner_code", "partner_name", "is_active"],
    "action": { "datatables": false, "createComposite": true, "updateComposite": true, "readComposite": true },
    "masterDetail": {
        "enabled": true,
        "transactionMode": "required",
        "details": [
            {
                "detailId": "vendor",
                "cardinality": "one",
                "detailTable": "vendor",
                "foreignKey": "business_partner_id",
                "cascadeDelete": false,
                "detailConfig": {
                    "tableName": "vendor",
                    "primaryKey": "vendor_id",
                    "fieldName": [
                        "vendor_id", "business_partner_id", "vendor_code",
                        "payment_term_days", "is_active"
                    ],
                    "requiredFields": ["vendor_code"]
                }
            },
            {
                "detailId": "customer",
                "cardinality": "one",
                "detailTable": "customer",
                "foreignKey": "business_partner_id",
                "cascadeDelete": false,
                "detailConfig": {
                    "tableName": "customer",
                    "primaryKey": "customer_id",
                    "fieldName": [
                        "customer_id", "business_partner_id", "customer_code",
                        "credit_limit", "payment_term_days", "is_active"
                    ],
                    "requiredFields": ["customer_code"]
                }
            },
            {
                "detailId": "contact",
                "cardinality": "many",
                "detailTable": "contact",
                "foreignKey": "business_partner_id",
                "cascadeDelete": true,
                "detailConfig": {
                    "tableName": "contact",
                    "primaryKey": "contact_id",
                    "fieldName": [
                        "contact_id", "business_partner_id", "contact_name", "contact_phone"
                    ],
                    "requiredFields": ["contact_name"]
                }
            }
        ]
    }
}
```

Setiap entri memakai properti yang sama dengan bentuk satu detail, ditambah `detailId` dan `cardinality`. Properti `enabled` dan `transactionMode` tetap berada di level atas `masterDetail`.

### Properti Entri `details[]`

| Property | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `detailId` | string | Ya | Nama key detail di body request dan response composite. Diawali huruf dan hanya berisi huruf, angka, atau underscore. Harus unik dalam satu RDF dan berbeda dari semua kolom di `fieldName` header |
| `cardinality` | string | Ya | `"one"` untuk detail dengan paling banyak satu baris per header, atau `"many"` untuk detail berisi banyak baris |
| `detailTable` | string | Ya | Nama tabel detail di database |
| `foreignKey` | string | Ya | Kolom FK di tabel detail yang merujuk ke header |
| `cascadeDelete` | boolean | Tidak | Sama seperti pada bentuk satu detail |
| `detailConfig` | object | Ya | Konfigurasi tabel detail, lihat [Properti `detailConfig`](#properti-detailconfig) |
| `headerCalculations` | object | Tidak | Agregat header dari baris detail. Hanya untuk `cardinality: "many"` |

### Kardinalitas Detail

Detail `many` mengikuti kontrak composite yang sudah ada: array pada `create-composite`, struktur `{insert, update, delete}` pada `update-composite`, dan array pada `read-composite`. Detail `one` berbentuk objek di ketiga endpoint itu. Kontraknya dijelaskan di [`create-composite`](../../api-spec/endpoint-create-composite.md#detail-satu-baris-one), [`update-composite`](../../api-spec/endpoint-update-composite.md#detail-satu-baris-one), dan [`read-composite`](../../api-spec/endpoint-read-composite.md#detail-satu-baris-one).

| Aspek | `one` | `many` |
|-------|-------|--------|
| Bentuk di `create-composite` | Objek, atau `null` | Array |
| Bentuk di `update-composite` | Objek, atau `null` untuk melepas detail | `{insert, update, delete}` |
| Bentuk di `read-composite` | Objek, atau `null` bila detail belum ada | Array |
| `headerCalculations` dan `autoCalculateFields` | Tidak didukung | Didukung |
| Key error constraint di response `400` dan `409` | `{detailId}.{kolom}` | Tanpa `details` |

Detail menjadi `one` bila kolom `foreignKey`-nya unique di SDF, yaitu unique yang hanya berisi kolom itu. Tabel detail tanpa unique tunggal pada `foreignKey` menjadi `many`. [`payload generate --detail`](../../commands/restforge-backend/payload/generate.md#master-detail-otomatis---detail) menurunkan nilai ini otomatis, sedangkan RDF yang ditulis manual boleh menetapkannya sendiri. Bila `cardinality: "one"` ditulis untuk kolom yang tidak unique, `endpoint create` mencetak peringatan karena database tidak menolak baris kedua.

### Aturan Validator

`payload validate` dan `endpoint create` menolak `details[]` yang melanggar salah satu aturan berikut.

| Kondisi | Contoh |
|---------|--------|
| `details` bukan array atau tidak punya entri | `"details": []` |
| `details` dicampur dengan bentuk satu detail | `detailTable`, `foreignKey`, `cascadeDelete`, `detailConfig`, atau `headerCalculations` ditulis di level atas `masterDetail` bersama `details` |
| Entri tidak punya `detailId`, `cardinality`, `detailTable`, `foreignKey`, atau `detailConfig` | Entri tanpa `foreignKey` |
| `cardinality` di luar `one` dan `many` | `"cardinality": "single"` |
| `detailId` bukan identifier, ganda, atau sama dengan kolom di `fieldName` header | `"detailId": "vendor-role"`, dua entri `vendor`, atau `detailId` `partner_name` |
| `headerCalculations` atau `detailConfig.autoCalculateFields` pada detail `one` | Entri `vendor` dengan `headerCalculations` |

Aturan di bagian [Deklarasi Invalid](#deklarasi-invalid) berlaku untuk setiap entri. Key di luar daftar properti entri tidak ditolak, tetapi menghasilkan warning.

### Bentuk Satu Detail Tetap Valid

RDF dengan `detailTable` dan `detailConfig` di level atas tidak perlu diubah. Bentuk itu dibaca sebagai satu detail `many` dengan `detailId` sama dengan nama tabel detail tanpa schema, sehingga key di body request dan response tetap seperti sebelumnya. Kedua bentuk tidak boleh dipakai bersamaan dalam satu RDF.

## Properti `detailConfig`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `tableName` | string | Ya | Nama tabel detail (umumnya sama dengan `masterDetail.detailTable`) |
| `primaryKey` | string | Ya | PK tabel detail. Wajib bertipe VARCHAR UUID-compatible |
| `fieldName` | array | Ya | Daftar kolom detail yang dikelola |
| `detailQuery` | string | Tidak | SQL SELECT untuk membaca detail. Mendukung inline atau `file:`. Kolom yang dipilih query ini, termasuk kolom JOIN, dikirim apa adanya di detail `/read-composite`. Bila tidak dideklarasikan, detail dibaca dengan query default sesuai [Kontrak Kolom Tabel Detail](#kontrak-kolom-tabel-detail) dan dibatasi ke `detailConfig.fieldName` |
| `requiredFields` | array | Tidak | Kolom yang wajib diisi saat insert detail item |
| `autoCalculateFields` | object | Tidak | Field detail yang dihitung otomatis per baris |
| `fieldValidation` | array | Tidak | Tipe dan constraint tiap kolom detail, format sama persis dengan [`fieldValidation`](./field-validation.md) level root RDF |
| `foreignKeys` | array | Tidak | Foreign key tabel detail yang bukan foreign key ke tabel master (lihat [Properti `detailConfig.foreignKeys[]`](#properti-detailconfigforeignkeys)) |
| `uniqueConstraints` | array | Tidak | Pemetaan unique constraint tabel detail, dengan format sama seperti [`uniqueConstraints`](./unique-constraints.md) level root. Pada detail `one` di MySQL dan Oracle, registry ini membuat error unique memuat key `{detailId}.{kolom}`. Terisi otomatis oleh `payload generate --detail` untuk bentuk `details[]` |
| `checkConstraints` | array | Tidak | Pemetaan CHECK constraint tabel detail, dengan format sama seperti [`checkConstraints`](./check-constraints.md) level root. Pada detail `one`, registry ini membuat pelanggaran CHECK memuat key `{detailId}.{kolom}`, dan tanpa registry response CHECK tidak memuat `details`. Terisi otomatis dari `checks` SDF oleh `payload generate --detail` untuk bentuk `details[]` |

`fieldValidation` dan `foreignKeys` bersifat aditif: RDF lama yang belum memilikinya tetap valid. Keduanya terisi otomatis oleh `payload generate --detail` dan dikonsumsi `payload migrate` untuk menghasilkan blok `details[]` UDF secara penuh (lihat [`payload migrate`](../../commands/restforge-backend/payload/migrate.md#master-detail-menjadi-details) dan [`catalogs/udf/master-detail.md`](../udf/master-detail.md)) — bukan properti yang perlu ditulis manual.

## Properti `detailConfig.foreignKeys[]`

Setiap entry mewakili satu foreign key tabel detail ke tabel selain tabel master (foreign key ke master sudah direpresentasikan lewat `masterDetail.foreignKey`, tidak diulang di sini).

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `column` | string | Nama kolom foreign key di tabel detail |
| `referencesTable` | string | Nama tabel yang dirujuk |
| `referencesColumn` | string | Nama kolom yang dirujuk di tabel tersebut |
| `displayColumn` | string atau `null` | Kolom tabel rujukan yang ditampilkan pada lookup detail. Lihat [`displayColumn`](#displaycolumn) |

```json
"foreignKeys": [
    {
        "column": "item_product_id",
        "referencesTable": "products",
        "referencesColumn": "product_id",
        "displayColumn": "name"
    }
]
```

### `displayColumn`

`payload generate --detail` menulis `displayColumn` setelah `referencesColumn`. Kolomnya dipilih dengan aturan yang sama seperti ekspansi [`--expand-fk`](../../commands/restforge-backend/payload/sync.md#ekspansi-foreign-key---expand-fk), yaitu kolom bernama `name` lebih dulu, lalu kolom teks yang ber-UNIQUE. Dengan begitu lookup di grid detail memakai kolom yang sama dengan halaman mandiri tabel rujukan.

Nilai `null` berarti tabel rujukan tidak punya kolom yang cocok. `payload migrate` lalu memakai primary key tabel rujukan sebagai tampilan lookup. Entri tanpa key `displayColumn`, misalnya pada RDF lama, tetap menghasilkan lookup yang sama seperti sebelumnya.

## Kontrak Kolom Tabel Detail

`line_number` bukan kolom wajib tersembunyi pada tabel detail. Kolom apa pun boleh dipakai untuk mengurutkan baris detail, ditentukan lewat dua jalur:

- Bila `detailConfig.fieldName` mencantumkan kolom bernama `line_number`, urutan default baris detail (dipakai ketika `detailQuery` tidak dideklarasikan) memakai kolom tersebut.
- Bila tidak, urutan default memakai `detailConfig.primaryKey`.

Seluruh kolom yang dicantumkan di `detailConfig.fieldName` wajib benar-benar ada di tabel detail. Lihat [Validasi Skema Detail vs Database](#validasi-skema-detail-vs-database) untuk pengecekan yang dijalankan `endpoint create`.

### Tujuan `line_number`: Urutan Bisnis Tersimpan

`line_number` menyimpan posisi item dalam satu header, terpisah dari primary key yang mengidentifikasi item. Gunakan kolom ini bila aplikasi membutuhkan susunan bisnis yang harus dipertahankan, misalnya urutan item penawaran yang disusun pengguna. Kolom ini bukan kewajiban untuk semua tabel transaksi: tabel tanpa `line_number` tetap dapat ditampilkan berurutan menggunakan `ORDER BY` primary key atau kolom lain. Nomor tampilan `#` sendiri tidak menyimpan pilihan urutan pengguna.

Posisi item dapat berubah, sehingga referensi permanen ke item harus memakai primary key, bukan `line_number`.

Versi saat ini belum menyediakan properti konfigurasi untuk mengatur ulang urutan baris detail, dan server belum mengisi `line_number` secara otomatis. Bila kolom `line_number` bersifat `NOT NULL` tanpa default, nilainya harus dikirim client di setiap item detail. Untuk halaman hasil generate, lihat [Menyimpan Urutan dengan Field Angka Manual](../udf/master-detail.md#menyimpan-urutan-dengan-field-angka-manual).

Urutan tersimpan harus digunakan ketika membaca kembali data. Query default mengikuti kontrak di atas; `detailQuery` kustom perlu `ORDER BY line_number` yang sesuai (gunakan alias tabel bila diperlukan).

## Properti `autoCalculateFields[name]`

`name` adalah kolom detail mana pun yang ada di `detailConfig.fieldName`.

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `type` | enum | `"calculated"` (dihitung application code lalu ikut SQL insert/update) atau `"generated"` (database GENERATED COLUMN, dikecualikan dari SQL) |
| `formula` | string | Ekspresi dengan referensi nama kolom, pola `kolomA * kolomB` (contoh: `qty_received * unit_price`) |
| `description` | string | Deskripsi opsional |

Untuk `type: "calculated"`, `formula` wajib ditulis manual mengikuti pola `kolomA * kolomB` persis (dua operand, keduanya kolom nyata di `detailConfig.fieldName`).

Untuk `type: "generated"`, `formula` biasanya terisi otomatis dari ekspresi kolom GENERATED di database saat memakai `payload generate --detail`, dan boleh mengandung cast tipe SQL (mis. `(qty)::numeric * unit_price`) — bentuk ini dinormalisasi otomatis saat dipakai untuk menghitung agregat header. Bila formula terlalu kompleks untuk dinormalisasi (mengandung `CASE`, fungsi, atau operand yang bukan kolom), agregat header yang menjumlahkan kolom tersebut tidak dihitung saat `createComposite`, dan `endpoint create` mencetak warning yang menyebut nama kolom dan alasannya.

## Properti `headerCalculations[name]`

Field header yang dihitung **oleh application code** dari detail items saat create/update composite. `name` wajib berupa kolom header yang sudah ada di `fieldName` level root RDF.

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `type` | enum | `"count"`, `"sum"`, `"avg"`, `"min"`, atau `"max"` |
| `source` | string | Referensi sumber. `items.length` untuk `count` (nilai `source` diabaikan untuk type ini). `items.<kolom-detail>` untuk `sum`/`avg`/`min`/`max`, wajib menunjuk kolom nyata di `detailConfig.fieldName` |
| `description` | string | Deskripsi opsional |

Semantik tiap `type`:

| `type` | Nilai dihitung |
|---|---|
| `count` | Jumlah baris detail |
| `sum` | Penjumlahan nilai `source` di seluruh baris detail |
| `avg` | Rata-rata nilai `source` di seluruh baris detail |
| `min` | Nilai `source` terkecil di antara baris detail |
| `max` | Nilai `source` terbesar di antara baris detail |

Kolom target `headerCalculations` tidak perlu dikirim di request `create-composite`; nilainya dihitung sebelum validasi field header berjalan, sehingga tidak perlu mengirim placeholder apa pun untuk kolom ini meski kolom tersebut ditandai `required` di `fieldValidation`.

Detail lengkap (concurrency contract, interaksi dengan `fieldPolicy`) dibahas di dokumentasi fitur CRUD composite terpisah.

## Deklarasi Invalid

`endpoint create` menolak generate (exit non-zero, tidak ada module yang ditulis) bila menemukan salah satu kondisi berikut pada `masterDetail.enabled: true`:

| Kondisi | Contoh |
|---|---|
| `headerCalculations[name].type` di luar lima nilai enum | `"type": "average"` (harus `avg`) |
| `headerCalculations[name].source` menunjuk kolom yang tidak ada di `detailConfig.fieldName` (untuk type selain `count`) | `"source": "items.kolom_tidak_ada"` |
| `headerCalculations[name]` bukan kolom header nyata di `fieldName` level root | Key salah ketik atau kolom yang belum dideklarasikan |
| `autoCalculateFields[name]` bukan kolom detail nyata di `detailConfig.fieldName` | Key salah ketik |

`masterDetail.enabled: true` tanpa `headerCalculations` sama sekali bukan error — blok ini opsional; tabel master-detail tanpa kebutuhan agregat header cukup mengosongkannya.

## Validasi Skema Detail vs Database

Saat `endpoint create` dijalankan untuk RDF dengan `masterDetail.enabled: true`, scope detail divalidasi terhadap skema database sungguhan, setelah validasi tabel header lolos. Empat pengecekan dijalankan:

| Pengecekan | Akibat bila gagal |
|---|---|
| Tabel `detailConfig.tableName` (atau `masterDetail.detailTable`) ada di database | Generate ditolak, tidak ada file ditulis |
| Setiap kolom di `detailConfig.fieldName` ada di tabel detail | Generate ditolak, pesan menyebut kolom yang hilang |
| `detailConfig.primaryKey` dan `masterDetail.foreignKey` ada di tabel detail | Generate ditolak, pesan menyebut kolom yang hilang |
| Kolom NOT NULL tanpa default di tabel detail yang tidak dicantumkan di `detailConfig.fieldName` (di luar kolom audit standar) | Warning, generate tetap lanjut |

Tiga pengecekan pertama bersifat blocking: ketidakcocokan menolak `endpoint create` sebelum satu file pun ditulis, dengan pesan yang menyebut nama tabel atau kolom yang bermasalah. Pengecekan keempat murni informasional, karena kolom tersebut berpotensi membuat insert detail gagal di runtime kecuali diisi mekanisme lain (default database, trigger, dan sejenisnya).

Pada bentuk `details[]`, keempat pengecekan dijalankan untuk setiap entri, dan hasilnya dicetak per `detailId`. Detail `one` yang kolom `foreignKey`-nya tidak punya unique tunggal di tabel menghasilkan peringatan, karena database tidak menolak baris detail kedua untuk header yang sama.

Validasi ini dilewati bersama validasi header ketika `endpoint create` dijalankan dengan `--skip-schema-check`.

## Field Tak Dikenal di Request Composite (Runtime)

Kolom yang lolos validasi di atas dijamin cocok dengan tabel detail sungguhan. Jaminan ini dipakai runtime `create-composite`/`update-composite`: field pada baris detail yang tidak termasuk kolom tabel detail (`detailConfig.fieldName` digabung PK detail, FK header, kolom `calculated`, dan kolom audit standar) ditolak dengan `400 Bad Request`. Pesan error menyebut nama field yang tidak dikenal dan daftar kolom valid tabel detail.

Seluruh operasi dalam request tersebut dibatalkan (transaksi di-rollback), termasuk baris header dan baris detail lain yang sudah diproses lebih dulu dalam request yang sama.

## Pengisian Manual Setelah Generate `--detail`

RDF hasil `payload generate --detail=<table>` memuat `detailConfig` terisi penuh dari introspeksi, termasuk `fieldValidation` dan `foreignKeys` (lihat [dokumentasi command](../../commands/restforge-backend/payload/generate.md#master-detail-otomatis---detail)), tetapi dua bagian berikut tetap perlu dilengkapi manual karena bersifat keputusan bisnis yang tidak bisa ditebak dari struktur database semata:

- `headerCalculations` — agregat header mana yang relevan untuk resource tersebut.
- `detailConfig.autoCalculateFields` dengan `type: "calculated"` — formula per baris yang dihitung application code (berbeda dari entry `type: "generated"` yang sudah terisi otomatis bila kolom detailnya adalah kolom GENERATED database).

RDF hasil generate menyertakan key `_manualStub` di level `masterDetail` berisi petunjuk singkat cara mengisi kedua bagian ini. Key tersebut murni dokumentasi diri (tidak pernah dibaca runtime) dan aman dihapus setelah `headerCalculations`/`autoCalculateFields` selesai dilengkapi manual.

Kedua bagian ini berada di level yang berbeda: `headerCalculations` sejajar dengan `detailConfig`,
langsung di dalam `masterDetail`, sedangkan `autoCalculateFields` satu level lebih dalam, yaitu di
dalam `detailConfig`. Karena keduanya diisi manual pada langkah yang sama, keliru meletakkan
`autoCalculateFields` sejajar dengan `detailConfig` (bukan di dalamnya) adalah kesalahan yang mudah
terjadi.

Pada bentuk `details[]`, kedua bagian ini ditulis di dalam entri `many` yang bersangkutan. `headerCalculations` sejajar dengan `detailConfig` entri itu, dan `autoCalculateFields` berada di dalam `detailConfig` entri itu.

Benar, `autoCalculateFields` di dalam `detailConfig`:

```json
{
    "masterDetail": {
        "detailConfig": {
            "autoCalculateFields": {
                "total_amount": { "type": "calculated", "formula": "qty_received * unit_price" }
            }
        },
        "headerCalculations": {
            "total_amount": { "type": "sum", "source": "items.total_amount" }
        }
    }
}
```

Salah, `autoCalculateFields` sejajar dengan `detailConfig`:

```json
{
    "masterDetail": {
        "detailConfig": {},
        "autoCalculateFields": {
            "total_amount": { "type": "calculated", "formula": "qty_received * unit_price" }
        },
        "headerCalculations": {
            "total_amount": { "type": "sum", "source": "items.total_amount" }
        }
    }
}
```

Penempatan salah di atas ditolak validasi dengan error yang menyebut lokasi yang benar; error ini
muncul baik saat `payload validate` maupun `endpoint create`, sehingga tidak berakhir sebagai
kalkulasi yang diabaikan diam-diam. Key lain yang tidak dikenal di dalam blok `masterDetail`
(selain `enabled`, `detailTable`, `foreignKey`, `cascadeDelete`, `transactionMode`, `detailConfig`,
`headerCalculations`, dan `_manualStub`) tidak ditolak, tetapi menghasilkan warning.

## Update Composite Detail-Only

Request `updateComposite` dengan body yang hanya berisi operasi detail (`insert`/`update`/`delete`) tanpa satu pun field header yang berubah adalah request valid — header tidak ikut diperbarui, tetapi seluruh kolom `headerCalculations` tetap dihitung ulang dari detail items yang tersisa setelah operasi selesai.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
