# Master Detail — `details[]`

> Pola halaman master-detail: satu page CRUD memiliki satu atau lebih tabel detail yang terkait.

## Konsep

Pola master-detail dipakai ketika satu record (master) memiliki kumpulan baris terkait (detail). Contoh klasik:

| Master | Detail |
|--------|--------|
| Sales Order | Order Item (banyak baris per order) |
| Invoice | Invoice Line Item |
| Stock Inbound | Stock Inbound Item |
| Purchase Order | PO Line Item |

UDF mendeklarasikan detail di field `details[]` di page object master. Setiap entry di `details[]` mendefinisikan satu tabel detail.

Detail punya dua kardinalitas. Detail `many` tampil sebagai grid berisi banyak baris, sedangkan detail `one` tampil sebagai sub-form untuk paling banyak satu baris per master, misalnya peran vendor dan peran customer pada Business Partner. Detail tanpa `cardinality` diperlakukan sebagai `many`.

Blok `details[]` boleh ditulis manual seperti contoh di bawah, tetapi umumnya dihasilkan otomatis oleh [`payload migrate`](../../commands/restforge-backend/payload/migrate.md#master-detail-menjadi-details) dari RDF backend ber-[`masterDetail`](../rdf/master-detail.md) — field, tipe, dan foreign key detail diturunkan langsung dari skema database lewat `payload generate --detail`, tanpa perlu ditulis tangan dari nol.

## Sintaks

```json
{
    "pageId": "sales-order",
    "pageTitle": "Sales Order",
    "primaryKey": "order_id",
    "displayField": "order_no",
    "apiPath": "sales-order",
    "fields": [
        {"name": "order_no", "label": "No Order", "type": "text", "required": true},
        {"name": "customer_id", "label": "Customer", "type": "select", "dataSource": {"type": "api", "resource": "customer"}}
    ],
    "details": [
        {
            "detailId": "items",
            "primaryKey": "item_id",
            "fields": [
                {"name": "product_id", "label": "Product", "type": "select", "dataSource": {"type": "api", "resource": "product"}},
                {"name": "qty", "label": "Qty", "type": "number"},
                {"name": "price", "label": "Price", "type": "number"}
            ]
        }
    ]
}
```

## Properti `details[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `detailId` | string | ✓ | Identifier detail (snake_case atau camelCase). Dipakai sebagai key di payload save dan URL endpoint detail |
| `cardinality` | string | ✗ | `"one"` atau `"many"`. Detail `one` dirender sebagai sub-form, sedangkan `many` atau tanpa `cardinality` dirender sebagai grid. Diisi oleh `payload migrate`, lihat [Detail Satu Baris](#detail-satu-baris-one) |
| `detailTitle` | string | ✗ | Judul section grid di form, juga dipakai di pesan validasi grid. Bila kosong, judul diambil dari `detailId` dalam Title Case, misalnya `good_receipt_item` menjadi `Good Receipt Item` |
| `primaryKey` | string | ✓ | Nama field primary key untuk baris detail |
| `fields` | array | ✓ | Array definisi field detail (minimal 1, sama struktur seperti field master) |
| `summary` | object | ✗ | Baris ringkasan agregat di bagian bawah grid detail (lihat [Properti `summary`](#properti-summary)) |
| `emptyText` | string | ✗ | Khusus detail `one`. Teks sub-form selama detail belum ada |
| `addLabel` | string | ✗ | Khusus detail `one`. Label tombol penambah detail |
| `removeLabel` | string | ✗ | Khusus detail `one`. Label tombol pelepas detail |
| `fieldRows` | array | ✗ | Khusus detail `one`. Susunan field di sub-form, dengan format sama seperti [`fieldRows`](./field-rows.md) page |

### Aturan Validasi `details[]`

| Aturan | Error |
|--------|-------|
| `detailId` wajib | `"[<pageId>].details[<i>] detailId must be provided"` |
| `primaryKey` wajib | `"[<pageId>].details[<detailId>] primaryKey must be provided"` |
| `fields` wajib non-empty | `"[<pageId>].details[<detailId>] fields must be provided and cannot be empty"` |
| Setiap field di `details[].fields[]` wajib `name`, `label`, `type` valid | Error per-field seperti di master |
| `cardinality` selain `one` dan `many` | Error: `"[<pageId>].details[<detailId>] cardinality must be 'one' or 'many', got <value>"` |
| `emptyText`, `addLabel`, atau `removeLabel` bukan string | Error: `"[<pageId>].details[<detailId>] <atribut> must be a string, got <tipe>"` |
| `emptyText`, `addLabel`, `removeLabel`, atau `fieldRows` pada detail yang bukan `one` | Warning, atribut diabaikan |
| `fieldRows` detail `one` bukan array | Error |
| `fieldRows` detail `one` menyebut field yang tidak ada, atau tidak menyebut field non-hidden | Warning, seperti `fieldRows` page |
| Field header bernama `{detailId}_{name}` bentrok dengan field detail `one` | Error, karena keduanya memakai id elemen `form-group-{detailId}-{name}` |
| `defaultValue` IdGen pada field detail melanggar aturan IdGen | Error, aturannya sama dengan field page (lihat [ID Generation](./id-generation.md)) |

## Field di Detail

Field di dalam `details[].fields[]` memakai struktur dan tipe yang sama dengan field master (lihat [`field-types.md`](./field-types.md) dan [`field-attributes.md`](./field-attributes.md)). Atribut umum (`required`, `inTable`, `tableOrder`, `placeholder`, `readonly`, dll.) berlaku penuh.

```json
"details": [
    {
        "detailId": "items",
        "primaryKey": "item_id",
        "fields": [
            {
                "name": "product_id",
                "label": "Produk",
                "type": "select",
                "required": true,
                "inTable": true,
                "tableOrder": 1,
                "tableField": "product_name",
                "dataSource": {
                    "type": "api",
                    "resource": "product",
                    "select": ["product_id", "product_name"]
                }
            },
            {
                "name": "qty",
                "label": "Qty",
                "type": "number",
                "required": true,
                "inTable": true,
                "tableOrder": 2,
                "min": 1
            },
            {
                "name": "price",
                "label": "Harga",
                "type": "number",
                "required": true,
                "inTable": true,
                "tableOrder": 3
            }
        ]
    }
]
```

Field `select` di dalam `details[].fields[]` mendukung empat atribut
`dataSource` tambahan — `lookupMode`, `lookupDisplay`, `optionColumns`, dan
`autofill` — untuk kebutuhan seperti mengisi harga otomatis begitu produk
dipilih, menampilkan lebih dari satu kolom informasi di dropdown, atau
mengganti dropdown dengan modal pencarian. Keempatnya berlaku juga di field
`select` halaman master (`pages[].fields[]`), dengan sejumlah batasan yang
tidak berlaku di grid detail. Lihat [Atribut Lookup Tambahan](./data-source.md#atribut-lookup-tambahan-autofill-optioncolumns-lookupdisplay)
di `data-source.md` untuk detail lengkap, contoh, dan batasan halaman master.

## Format Tampilan Angka (`format` dan `decimalPlaces`)

Field detail bertipe `number` — baik yang bisa diedit langsung maupun hasil kalkulasi otomatis (lihat [Kalkulasi Otomatis per Baris](#kalkulasi-otomatis-per-baris-calculated) di bawah) — mendukung dua atribut tambahan untuk memformat tampilan angkanya di grid:

```json
{
    "name": "price",
    "label": "Price",
    "type": "number",
    "required": true,
    "inTable": true,
    "tableOrder": 3,
    "format": "currency",
    "decimalPlaces": 2
}
```

| Atribut | Tipe | Keterangan |
|---------|------|-----------|
| `format` | string | Format tampilan angka. Valid: `"number"` (pemisah ribuan biasa) atau `"currency"` (format mata uang). Nilai lain diperlakukan sama dengan `"number"` |
| `decimalPlaces` | integer | Jumlah digit desimal yang ditampilkan. Default `0` bila tidak diisi |
| `minDecimalPlaces` | integer | Jumlah digit desimal minimum yang selalu tampil. Nol di belakang koma setelah digit minimum dibuang. Wajib disertai `decimalPlaces` dan tidak boleh melebihinya |

Atribut `format`, `decimalPlaces`, dan `minDecimalPlaces` hanya memengaruhi tampilan input dan sel tabel grid detail. Nilai yang dikirim ke server dibaca dari teks tampilan. Nilai itu tidak dibulatkan selama `decimalPlaces` sama dengan skala kolom database. Kondisi ini terjamin bila `decimalPlaces` berasal dari `payload migrate`, bukan diisi manual dengan nilai yang lebih kecil dari skala kolom.

Sel angka di grid mengikuti aturan input angka di halaman master (lihat [`field-types.md`](./field-types.md#number)). Angka 0 tetap 0, sel kosong dikirim sebagai `null`, dan tanda minus ditolak pada field dengan `min` bernilai 0 atau lebih. Kalkulasi baris dan total menghitung sel kosong sebagai 0.

## Nilai Awal Baris Baru (`defaultValue`)

Field detail bertipe `number` dapat memakai `defaultValue` untuk menentukan
nilai awal setiap kali baris baru ditambahkan ke grid — bukan sekadar nilai
default form biasa (lihat [`field-attributes.md`](./field-attributes.md#defaultvalue-khusus)
untuk bentuk literal maupun idgen):

```json
{
    "name": "quantity",
    "label": "Qty",
    "type": "number",
    "required": true,
    "inTable": true,
    "tableOrder": 2,
    "defaultValue": 1
}
```

| Kondisi | Nilai Baris Baru |
|---------|------------------|
| `defaultValue` berupa angka atau string angka | Nilai tersebut |
| `defaultValue` tidak diisi, atau berisi nilai non-numerik | Kosong (`null`), kecuali field `calculated` yang dimulai dari `0` |

Perilaku ini hanya berlaku saat baris ditambahkan lewat tombol tambah baris.
Baris yang dimuat dari data existing (mode edit, lewat `read-composite`)
selalu menampilkan nilai tersimpan apa adanya — tidak pernah ditimpa
`defaultValue`. Field `checkbox` dan opsi pertama `select` static di grid
detail tetap mengikuti aturan `defaultValue` umum, tidak berubah oleh
perilaku ini.

Untuk mengisi nilai awal dari kolom tabel master lookup (mis. harga produk
mengikuti produk yang dipilih), pakai `dataSource.autofill`, bukan
`defaultValue` — lihat [Atribut Lookup Tambahan](./data-source.md#atribut-lookup-tambahan-autofill-optioncolumns-lookupdisplay).
Kedua mekanisme melayani sumber nilai yang berbeda: `defaultValue` untuk
konstanta yang sama di setiap baris baru (mis. quantity selalu mulai dari 1),
`autofill` untuk nilai yang bergantung pada opsi lookup yang dipilih user
(mis. harga produk).

## Lebar Kolom Grid (`width`)

Header grid detail mendukung atribut `width` yang sama dengan field halaman
master (lihat [`field-attributes.md`](./field-attributes.md#atribut-umum-berlaku-untuk-semua-tipe)),
dengan default yang disesuaikan untuk tabel grid:

| Kolom | Lebar |
|-------|-------|
| Nomor urut (`#`) | `40px`, tetap |
| Field ber-`width` eksplisit | Nilai `width` apa adanya |
| Field `number` tanpa `width` | `130px` |
| Field tipe lain tanpa `width` | Tanpa lebar tetap, berbagi sisa ruang tabel |
| Kolom aksi (hapus baris) | `50px`, tetap |

```json
[
    {"name": "product_id", "label": "Product", "type": "select", "width": "45%"},
    {"name": "quantity", "label": "Qty", "type": "number"},
    {"name": "price", "label": "Price", "type": "number", "width": "150px"}
]
```

Pada contoh di atas, kolom `Product` memakai lebar eksplisit `45%`, kolom
`Price` memakai lebar eksplisit `150px`, sedangkan kolom `Qty` (tipe `number`
tanpa `width`) mendapat `130px` secara otomatis.

## Nomor Urut Baris (`line_number`)

Kolom pertama grid detail (`#`) menampilkan nomor urut baris. Nomor ini
dihitung di browser: baris baru mendapat nomor berikutnya, dan seluruh baris
dinomori ulang setiap kali satu baris dihapus. Nomor urut tidak berasal dari
UDF dan tidak dikirim ke server. Item detail pada body `create-composite` dan
`update-composite` hanya memuat field yang dideklarasikan di
`details[].fields[]`, ditambah `primaryKey` detail untuk baris yang sudah
tersimpan. Backend menolak key item detail di luar `detailConfig.fieldName`
RDF, sehingga tabel detail tidak perlu memiliki kolom `line_number` (lihat
[Kontrak Kolom Tabel Detail](../rdf/master-detail.md#kontrak-kolom-tabel-detail)).

### Tujuan dan Status Pengaturan Urutan

`#` adalah nomor tampilan, sedangkan `line_number` menyimpan urutan bisnis per header. Tabel tanpa kolom tersebut tetap dapat ditampilkan terurut berdasarkan primary key atau kolom lain. Alasan menggunakan `line_number` adalah kebutuhan menyimpan susunan item yang dipilih pengguna, bukan sekadar menampilkan angka berurutan.

Versi saat ini belum menyediakan pengaturan urutan baris detail lewat UI, misalnya drag-and-drop. Server juga belum mengisi `line_number` secara otomatis. Akibatnya, tabel detail dengan kolom `line_number` `NOT NULL` tanpa default gagal disimpan dari halaman hasil generate bila field tersebut tidak dideklarasikan di UDF. Bila aplikasi tidak memerlukan urutan bisnis tersimpan, tidak perlu menambahkan `line_number`.

### Menyimpan Urutan dengan Field Angka Manual

Bila tabel detail memang memiliki kolom `line_number` dan urutannya perlu
tersimpan, deklarasikan sebagai field detail:

```json
{"name": "line_number", "label": "No", "type": "number"}
```

Dengan deklarasi itu, `line_number` ikut dikirim satu kali per item dan
nilainya mengikuti nomor urut grid: nilai tersimpan saat baris dimuat lewat
`read-composite` (atau urutan baris bila kolomnya kosong), nomor berikutnya
saat baris ditambah, lalu dinomori ulang setelah hapus. `defaultValue` tidak
berlaku untuk field ini. Field tampil sebagai kolom input angka biasa di
samping kolom `#`, mengikuti atribut tampilan field seperti `width` dan
`format`. Di sisi backend, kolom `line_number` yang ada di
`detailConfig.fieldName` dipakai sebagai urutan default baris detail.

Cara ini punya dua konsekuensi. Grid menampilkan kolom input `No` di samping
kolom `#`. Deklarasi manual tetap ada saat `payload migrate` dijalankan ulang, tetapi
hilang bila `--overwrite` dipakai, sehingga field perlu ditambahkan kembali setelahnya.

## Kalkulasi Otomatis per Baris (`calculated`)

Field detail dapat menghitung nilainya sendiri dari field lain di baris yang sama, lewat kombinasi atribut `readonly` dan `calculated.formula`:

```json
{
    "name": "subtotal",
    "label": "Subtotal",
    "type": "number",
    "readonly": true,
    "calculated": { "formula": "qty * price" },
    "format": "number"
}
```

| Atribut | Tipe | Keterangan |
|---------|------|-----------|
| `calculated.formula` | string | Formula kalkulasi. **Hanya mendukung pola perkalian dua field**, `"<fieldA> * <fieldB>"` — kedua nama field harus ada di `fields[]` detail yang sama. Formula di luar pola ini (penjumlahan, pengurangan, pembagian, lebih dari dua operand, fungsi) tidak didukung runtime saat ini |

Nilai field ber-`calculated` dihitung ulang otomatis setiap kali salah satu field operand formula (`qty`/`price` pada contoh di atas) berubah di baris yang sama, tanpa reload halaman. Karena nilainya selalu diturunkan otomatis, field ini sebaiknya disertai `readonly: true` (lihat [`field-attributes.md`](./field-attributes.md#atribut-umum-berlaku-untuk-semua-tipe)) agar tidak bisa diedit manual. `format`/`decimalPlaces` (lihat di atas) berlaku sama untuk field hasil kalkulasi.

## Properti `summary`

Blok opsional pada `details[]` yang menampilkan baris ringkasan di bagian bawah grid detail. Nilainya dihitung ulang otomatis setiap kali baris detail ditambah, diubah, atau dihapus, tanpa reload halaman.

```json
"summary": {
    "totalItems": true,
    "totalQtyField": "qty",
    "grandTotalField": "subtotal"
}
```

| Properti | Tipe | Keterangan |
|----------|------|-----------|
| `totalItems` | boolean | Jika `true`, tampilkan jumlah baris detail yang sedang ada di grid |
| `totalQtyField` | string | Nama field detail (harus ada di `fields[]` detail yang sama) yang nilainya dijumlahkan lintas baris dan ditampilkan sebagai total kuantitas |
| `grandTotalField` | string | Nama field detail (biasanya field ber-`calculated`, lihat di atas) yang nilainya dijumlahkan lintas baris dan ditampilkan sebagai grand total |

Tampilan `totalQtyField` dan `grandTotalField` mengikuti `format`/`decimalPlaces` field yang dirujuk ([Format Tampilan Angka](#format-tampilan-angka-format-dan-decimalplaces) di atas), sehingga baris ringkasan konsisten dengan kolom dan input di atasnya.

Ketiga properti independen — boleh mengisi salah satu saja, kombinasi mana pun, atau mengosongkan seluruh blok `summary` bila grid detail tidak perlu baris ringkasan.

## Detail Satu Baris (`one`)

Detail `one` cocok untuk data yang melengkapi master dengan paling banyak satu baris, seperti peran vendor dan peran customer pada satu Business Partner. Grid kurang tepat untuk kasus ini karena tidak bisa membedakan peran yang belum ada dari peran yang ada tetapi masih kosong, dan tidak menyediakan alur tambah atau lepas peran.

Detail `one` ditulis sebagai entry `details[]` dengan `cardinality: "one"`. Contoh berikut memuat page Business Partner dengan dua detail `one` yang ditempatkan di tab masing-masing, serta dua kolom list `listOnly`.

```json
{
    "pages": [
        {
            "pageId": "business-partner",
            "pageTitle": "Business Partner",
            "apiPath": "business-partner",
            "primaryKey": "business_partner_id",
            "displayField": "partner_name",
            "editorType": "modal",
            "fields": [
                {
                    "name": "partner_code",
                    "label": "Partner Code",
                    "type": "text",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1
                },
                {
                    "name": "partner_name",
                    "label": "Partner Name",
                    "type": "text",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 2
                },
                {
                    "name": "vendor_code",
                    "label": "Vendor",
                    "type": "text",
                    "listOnly": true,
                    "inTable": true,
                    "tableOrder": 3
                },
                {
                    "name": "customer_code",
                    "label": "Customer",
                    "type": "text",
                    "listOnly": true,
                    "inTable": true,
                    "tableOrder": 4
                }
            ],
            "details": [
                {
                    "detailId": "vendor",
                    "cardinality": "one",
                    "detailTitle": "Vendor",
                    "primaryKey": "vendor_id",
                    "emptyText": "This partner is not a vendor yet. Add the vendor role to buy from this partner.",
                    "addLabel": "Add Vendor Role",
                    "removeLabel": "Remove Vendor Role",
                    "fields": [
                        {
                            "name": "vendor_code",
                            "label": "Vendor Code",
                            "type": "text",
                            "required": true,
                            "readonlyOnEdit": true,
                            "defaultValue": {
                                "source": "idgen",
                                "mode": "number",
                                "resource": "vendor",
                                "format": "text",
                                "prefix": "VEND",
                                "separator": "-",
                                "numDigits": 3
                            }
                        },
                        {
                            "name": "payment_term_days",
                            "label": "Payment Term Days",
                            "type": "number"
                        },
                        {
                            "name": "is_active",
                            "label": "Status",
                            "type": "checkbox",
                            "defaultValue": true
                        }
                    ],
                    "fieldRows": [
                        { "fields": ["vendor_code", "payment_term_days"] },
                        { "fields": ["is_active"] }
                    ]
                },
                {
                    "detailId": "customer",
                    "cardinality": "one",
                    "detailTitle": "Customer",
                    "primaryKey": "customer_id",
                    "fields": [
                        {
                            "name": "customer_code",
                            "label": "Customer Code",
                            "type": "text",
                            "required": true
                        },
                        {
                            "name": "credit_limit",
                            "label": "Credit Limit",
                            "type": "number",
                            "format": "number",
                            "decimalPlaces": 2
                        },
                        {
                            "name": "payment_term_days",
                            "label": "Payment Term Days",
                            "type": "number"
                        }
                    ]
                }
            ],
            "formLayout": {
                "tabs": [
                    {
                        "title": "General",
                        "rows": [
                            {
                                "columns": [
                                    {
                                        "cards": [
                                            {
                                                "title": "Identity",
                                                "fieldRows": [
                                                    { "fields": ["partner_code", "partner_name"] }
                                                ]
                                            }
                                        ]
                                    }
                                ]
                            }
                        ]
                    },
                    {
                        "title": "Vendor",
                        "rows": [
                            { "columns": [ { "cards": [ { "detail": "vendor" } ] } ] }
                        ]
                    },
                    {
                        "title": "Customer",
                        "rows": [
                            { "columns": [ { "cards": [ { "detail": "customer" } ] } ] }
                        ]
                    }
                ]
            }
        }
    ]
}
```

### Asal `cardinality`

`payload migrate` menurunkan `cardinality` dari RDF ke setiap entry `details[]`. Di RDF, detail menjadi `one` bila kolom penghubungnya unique di SDF, seperti dijelaskan di [Kardinalitas Detail](../rdf/master-detail.md#kardinalitas-detail). RDF bentuk lama menghasilkan entry tanpa key `cardinality`, sehingga UDF yang sudah ada tidak berubah.

Migrate ulang selalu mengikuti RDF untuk `cardinality`, termasuk bila nilainya diubah manual di UDF. Atribut `emptyText`, `addLabel`, `removeLabel`, dan `fieldRows` hasil penyuntingan tetap dipertahankan. Field detail `one` tidak diberi `inTable` dan `tableOrder`, karena detail `one` tidak punya grid.

### Tampilan Sub-Form

Sub-form punya dua keadaan. Selama detail belum ada, sub-form menampilkan `emptyText` dan tombol `addLabel`. Setelah tombol ditekan, sub-form menampilkan field detail dan tombol `removeLabel`.

Atribut teks yang tidak diisi disusun dari `detailTitle`.

| Atribut | Teks bawaan |
|---------|-------------|
| `emptyText` | `No {detailTitle} has been added yet.` |
| `addLabel` | `Add {detailTitle}` |
| `removeLabel` | `Remove {detailTitle}` |

`fieldRows` mengatur susunan field di sub-form dengan format yang sama seperti [`fieldRows`](./field-rows.md) page. Tanpa `fieldRows`, field tampil satu per baris sesuai urutan di `fields[]`. Pada page ber-`fieldLayout: "horizontal"`, `fieldRows` detail diabaikan.

### Perilaku di Form

Sub-form mengikuti perilaku form yang sudah ada.

- Load record mengisi sub-form dari `read-composite`, dan detail yang bernilai `null` menampilkan keadaan kosong.
- Menambah atau melepas detail dihitung sebagai perubahan isian, sehingga konfirmasi Discard changes ikut muncul.
- Validasi field detail hanya berjalan selama detail ada.
- Saat simpan, detail yang ada dikirim sebagai objek dan detail tersimpan yang dilepas dikirim sebagai `null`. Detail yang tidak disentuh tidak dikirim. Kontrak lengkapnya ada di [`create-composite`](../../api-spec/endpoint-create-composite.md#detail-satu-baris-one) dan [`update-composite`](../../api-spec/endpoint-update-composite.md#detail-satu-baris-one).
- `readonlyOnEdit` pada field detail mengunci field setelah detail tersimpan.
- Field detail yang ber-`defaultValue` IdGen meminta nilai saat detail ditambahkan (lihat [ID Generation](./id-generation.md#idgen-pada-field-detail-one)).
- Pada mode duplicate, field detail `unique` mendapat akhiran `-COPY`, field IdGen meminta nilai baru, dan primary key detail tidak disalin.

### Id Elemen dan Key Error

Field detail `one` memakai id elemen `form-group-{detailId}-{name}`, dengan underscore pada nama menjadi tanda hubung. Field `payment_term_days` pada detail `vendor`, misalnya, memakai id `form-group-vendor-payment-term-days`. Namespace ini mencegah dua detail yang punya kolom bernama sama saling bentrok.

Key `{detailId}.{kolom}` pada response error `400` dan `409` dipetakan ke field sub-form yang bersangkutan, dan tab yang memuat card detail itu menampilkan badge jumlah error. Error yang tidak menunjuk field yang dirender, seperti penolakan unique untuk detail kedua, tampil di tingkat sub-form.

### Detail di Dalam Card `formLayout`

Card di `formLayout` dapat merujuk satu detail lewat `"detail": "<detailId>"`, sebagai pengganti `fieldRows` atau `rows`. Card detail berlaku untuk detail `one` maupun `many`, dan hanya boleh berada di card tingkat pertama. Satu `detailId` paling banyak dirujuk satu card, dan detail yang tidak dirujuk card mana pun dirender setelah seluruh `formLayout`. Aturan lengkapnya ada di user guide designer pada bagian Detail di dalam Card.

### Kolom Khusus List (`listOnly`)

Kolom hasil JOIN di `datatablesQuery`, seperti `vendor_code` dan `customer_code` pada contoh di atas, perlu tampil di tabel list tanpa ikut form. Field seperti ini diberi `listOnly: true`, dengan aturan di [`field-attributes.md`](./field-attributes.md#listonly).

## Multiple Details Per Master

Satu master dapat memiliki beberapa tabel detail:

```json
{
    "pageId": "purchase-order",
    "details": [
        {
            "detailId": "items",
            "primaryKey": "po_item_id",
            "fields": [ /* ... */ ]
        },
        {
            "detailId": "attachments",
            "primaryKey": "attachment_id",
            "fields": [ /* ... */ ]
        },
        {
            "detailId": "approvals",
            "primaryKey": "approval_id",
            "fields": [ /* ... */ ]
        }
    ]
}
```

Setiap detail di-render sebagai bagian terpisah di form master. Tombol tambah baris pada tiap grid detail berlabel `Add Item` dan memakai gaya sekunder bergaris tepi, sehingga tombol penyimpan form master tetap satu-satunya tombol primer di panel.

Detail `many` dan detail `one` boleh dicampur dalam satu master. Detail `one` dirender sebagai sub-form dan dapat ditempatkan di tab lewat card `detail` di `formLayout`, sedangkan detail yang tidak dirujuk card mana pun dirender setelah seluruh `formLayout`.

## Endpoint API yang Dipakai

Generator menggunakan pola endpoint berikut untuk operasi detail:

| Operasi | Endpoint |
|---------|----------|
| Load detail per master | `GET <apiBaseUrl>/<apiPath>/<masterId>/<detailId>` |
| Save master + details (composite) | `POST <apiBaseUrl>/<apiPath>` dengan body `{master: {...}, <detailId>: [...]}`. Detail `one` dikirim sebagai objek atau `null` |
| Update master + details | `PUT <apiBaseUrl>/<apiPath>/<masterId>` dengan body composite |

Detail komposisi payload save dan response struktur ditangani oleh plugin runtime. Lihat dokumentasi plugin untuk format payload spesifik.

## Aturan dan Batasan

| Aspek | Behavior |
|-------|----------|
| Jumlah detail per master | Tidak dibatasi validator. Plugin runtime bisa memberi batasan praktis |
| Nested detail (detail dari detail) | Tidak didukung. Hanya 1 level detail per master |
| Detail tanpa master | Tidak didukung. Detail selalu terkait master |
| `detailId` unik per page | Tidak ada validasi unik secara eksplisit di validator. Tetapi pastikan unik untuk menghindari konflik save |

## Contoh Konfigurasi Lengkap

### Sales Order dengan Order Item

```json
{
    "pageId": "sales-order",
    "pageTitle": "Sales Order",
    "pageIcon": "handcart",
    "apiPath": "sales-order",
    "primaryKey": "order_id",
    "displayField": "order_no",
    "features": {
        "enableSearch": true,
        "fieldLayout": "vertical"
    },
    "fields": [
        {
            "name": "order_no",
            "label": "No Order",
            "type": "text",
            "required": true,
            "inTable": true,
            "tableOrder": 1,
            "readonly": true,
            "defaultValue": {
                "source": "idgen",
                "mode": "number",
                "resource": "sales-order",
                "format": "yyyymm",
                "numDigits": 5,
                "separator": "/",
                "reserve": true,
                "ttl": 60
            }
        },
        {
            "name": "order_date",
            "label": "Tanggal",
            "type": "date",
            "required": true,
            "inTable": true,
            "tableOrder": 2
        },
        {
            "name": "customer_id",
            "label": "Customer",
            "type": "select",
            "required": true,
            "inTable": true,
            "tableOrder": 3,
            "tableField": "customer_name",
            "dataSource": {
                "type": "api",
                "resource": "customer",
                "select": ["customer_id", "customer_name"]
            }
        }
    ],
    "fieldRows": [
        {"fields": ["order_no", "order_date"]},
        {"fields": ["customer_id"]}
    ],
    "details": [
        {
            "detailId": "items",
            "primaryKey": "order_item_id",
            "fields": [
                {
                    "name": "product_id",
                    "label": "Produk",
                    "type": "select",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1,
                    "tableField": "product_name",
                    "dataSource": {
                        "type": "api",
                        "resource": "product",
                        "select": ["product_id", "product_name"]
                    }
                },
                {
                    "name": "qty",
                    "label": "Qty",
                    "type": "number",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 2,
                    "min": 1
                },
                {
                    "name": "price",
                    "label": "Harga",
                    "type": "number",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 3
                }
            ]
        }
    ]
}
```

---

← [`features.md`](./features.md) | [Selanjutnya: `workflow-actions.md`](./workflow-actions.md) →
