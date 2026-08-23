# Field Master Detail (`masterDetail`)

Konfigurasi untuk operasi composite (header + detail dalam satu transaksi). Wajib ditetapkan jika action `createComposite`, `updateComposite`, atau `readComposite` aktif.

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

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `enabled` | boolean | Mengaktifkan fitur master-detail |
| `detailTable` | string | Nama tabel detail di database |
| `foreignKey` | string | Kolom FK di tabel detail yang merujuk ke header |
| `cascadeDelete` | boolean | Jika `true`, hapus header menghapus seluruh detail items |
| `transactionMode` | string | Hanya nilai `"required"` yang didukung. Semua operasi dalam satu transaction |
| `detailConfig` | object | Konfigurasi tabel detail (lihat di bawah) |
| `headerCalculations` | object | Field header yang dihitung secara otomatis dari detail items |

Blok `masterDetail` beserta `detailConfig` dapat digenerate otomatis dari introspeksi database lewat [`payload generate --detail=<table>`](../../commands/restforge-backend/payload/generate.md#master-detail-otomatis---detail). `headerCalculations` dan entry `autoCalculateFields` bertipe `calculated` tetap perlu dilengkapi manual setelahnya; lihat bagian [Pengisian Manual Setelah Generate](#pengisian-manual-setelah-generate---detail) di bawah.

## Properti `detailConfig`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `tableName` | string | Ya | Nama tabel detail (umumnya sama dengan `masterDetail.detailTable`) |
| `primaryKey` | string | Ya | PK tabel detail. Wajib bertipe VARCHAR UUID-compatible |
| `fieldName` | array | Ya | Daftar kolom detail yang dikelola |
| `detailQuery` | string | Ya | SQL SELECT untuk membaca detail. Mendukung inline atau `file:` |
| `requiredFields` | array | Tidak | Kolom yang wajib diisi saat insert detail item |
| `autoCalculateFields` | object | Tidak | Field detail yang dihitung otomatis per baris |
| `fieldValidation` | array | Tidak | Tipe dan constraint tiap kolom detail, format sama persis dengan [`fieldValidation`](./field-validation.md) level root RDF |
| `foreignKeys` | array | Tidak | Foreign key tabel detail yang bukan foreign key ke tabel master (lihat [Properti `detailConfig.foreignKeys[]`](#properti-detailconfigforeignkeys)) |

`fieldValidation` dan `foreignKeys` bersifat aditif: RDF lama yang belum memilikinya tetap valid. Keduanya terisi otomatis oleh `payload generate --detail` dan dikonsumsi `payload migrate` untuk menghasilkan blok `details[]` UDF secara penuh (lihat [`payload migrate`](../../commands/restforge-backend/payload/migrate.md#master-detail-menjadi-details) dan [`catalogs/udf/master-detail.md`](../udf/master-detail.md)) — bukan properti yang perlu ditulis manual.

## Properti `detailConfig.foreignKeys[]`

Setiap entry mewakili satu foreign key tabel detail ke tabel selain tabel master (foreign key ke master sudah direpresentasikan lewat `masterDetail.foreignKey`, tidak diulang di sini).

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `column` | string | Nama kolom foreign key di tabel detail |
| `referencesTable` | string | Nama tabel yang dirujuk |
| `referencesColumn` | string | Nama kolom yang dirujuk di tabel tersebut |

```json
"foreignKeys": [
    { "column": "item_product_id", "referencesTable": "products", "referencesColumn": "product_id" }
]
```

## Kontrak Kolom Tabel Detail

`line_number` bukan kolom wajib tersembunyi pada tabel detail. Kolom apa pun boleh dipakai untuk mengurutkan baris detail, ditentukan lewat dua jalur:

- Bila `detailConfig.fieldName` mencantumkan kolom bernama `line_number`, urutan default baris detail (dipakai ketika `detailQuery` tidak dideklarasikan) memakai kolom tersebut.
- Bila tidak, urutan default memakai `detailConfig.primaryKey`.

Seluruh kolom yang dicantumkan di `detailConfig.fieldName` wajib benar-benar ada di tabel detail. Lihat [Validasi Skema Detail vs Database](#validasi-skema-detail-vs-database) untuk pengecekan yang dijalankan `endpoint create`.

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
