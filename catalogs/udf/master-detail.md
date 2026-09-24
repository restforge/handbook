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
| `primaryKey` | string | ✓ | Nama field primary key untuk baris detail |
| `fields` | array | ✓ | Array definisi field detail (minimal 1, sama struktur seperti field master) |
| `summary` | object | ✗ | Baris ringkasan agregat di bagian bawah grid detail (lihat [Properti `summary`](#properti-summary)) |

### Aturan Validasi `details[]`

| Aturan | Error |
|--------|-------|
| `detailId` wajib | `"[<pageId>].details[<i>] detailId must be provided"` |
| `primaryKey` wajib | `"[<pageId>].details[<detailId>] primaryKey must be provided"` |
| `fields` wajib non-empty | `"[<pageId>].details[<detailId>] fields must be provided and cannot be empty"` |
| Setiap field di `details[].fields[]` wajib `name`, `label`, `type` valid | Error per-field seperti di master |

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

Kedua atribut ini murni memengaruhi tampilan (input dan sel tabel grid detail). Nilai yang disimpan dan dikirim ke server adalah hasil `parseNumber` dari teks tampilan, sehingga tidak dibulatkan hanya bila `decimalPlaces` sama dengan skala kolom database, kondisi yang terjamin bila `decimalPlaces` berasal dari `payload migrate` dan bukan diisi manual dengan nilai yang lebih kecil dari skala kolom.

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
| `defaultValue` tidak diisi, atau berisi nilai non-numerik | `0` |

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
kolom `#`. Deklarasi manual juga hilang saat `payload migrate` dijalankan ulang
dengan `--overwrite`, sehingga field perlu ditambahkan kembali setelahnya.

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

## Endpoint API yang Dipakai

Generator menggunakan pola endpoint berikut untuk operasi detail:

| Operasi | Endpoint |
|---------|----------|
| Load detail per master | `GET <apiBaseUrl>/<apiPath>/<masterId>/<detailId>` |
| Save master + details (composite) | `POST <apiBaseUrl>/<apiPath>` dengan body `{master: {...}, <detailId>: [...]}` |
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
