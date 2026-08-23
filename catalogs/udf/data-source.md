# Sumber Data — `dataSource`

> Konfigurasi sumber data untuk dropdown `select` dan filter `dataFilters[]`.

## Dua Tipe Sumber Data

Generator mendukung dua tipe sumber data:

| Tipe | `dataSource.type` | Cocok untuk |
|------|-------------------|-------------|
| Static | `"static"` | Data tetap, jumlah sedikit (<10 opsi) |
| API | `"api"` | Data dinamis, jumlah banyak, sering berubah |

Struktur `dataSource` yang sama dipakai di field bertipe `select` dan di `dataFilters[]` (filter di toolbar).

## Static Source

Opsi dropdown didefinisikan langsung di payload sebagai array.

```json
{
    "name": "contact_type",
    "label": "Tipe Kontak",
    "type": "select",
    "required": true,
    "dataSource": {
        "type": "static",
        "options": [
            {"value": "personal", "text": "Personal"},
            {"value": "business", "text": "Business"},
            {"value": "family", "text": "Family"}
        ]
    }
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `type` | string | ✓ | Harus bernilai `"static"` |
| `options` | array | ✓ | Array objek `{value, text}` |
| `options[].value` | string | ✓ | Nilai yang disimpan ke database |
| `options[].text` | string | ✓ | Teks yang ditampilkan di dropdown |

### Perilaku Select Required vs Optional

| Kondisi | Behavior |
|---------|----------|
| `required: true` | Opsi pertama langsung terpilih, tanpa placeholder |
| `required: false` (default) | Menampilkan placeholder `-- Select --` dengan value kosong sebagai opsi pertama |

## API Source

Opsi dropdown dimuat secara dinamis dari endpoint backend lewat AJAX saat halaman dimuat.

```json
{
    "name": "city_id",
    "label": "Kota",
    "type": "select",
    "inTable": true,
    "tableOrder": 4,
    "tableField": "city_name",
    "dataSource": {
        "type": "api",
        "resource": "city",
        "select": ["city_id", "city_name"]
    }
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `type` | string | ✓ | Harus bernilai `"api"` |
| `resource` | string | ✓ | Nama resource backend. Dipakai di pola endpoint `POST /<apiBaseUrl>/<resource>/lookup` |
| `select` | array of string | ✗ | Nama kolom yang di-request ke API lookup |
| `url` | string | ✗ | Endpoint lookup eksplisit. Perilakunya berbeda antara field select di halaman master dan field select di grid detail master-detail — lihat [Atribut `url` dan Endpoint Lookup](#atribut-url-dan-endpoint-lookup) |

### Atribut `url` dan Endpoint Lookup

Endpoint lookup yang di-request selalu mengikuti pola `<resource>/lookup`. Bila `url` tidak diisi, endpoint di-default ke pola tersebut. Cara `url` diperlakukan berbeda tergantung lokasi field:

| Lokasi field | Perilaku |
|---|---|
| Field `select` di halaman master, termasuk field cascade (`dependsOn`) | Endpoint lookup selalu `<resource>/lookup`. Atribut `url` tidak berpengaruh sama sekali meski diisi |
| Field `select` di grid detail master-detail (`details[].fields[]`, lihat [`catalogs/udf/master-detail.md`](./master-detail.md)) | `url` eksplisit dipakai apa adanya bila diisi. Bila kosong atau tidak ada, endpoint di-default `<resource>/lookup` |

Dalam praktik, mengosongkan `url` menghasilkan endpoint yang sama (`<resource>/lookup`) di kedua lokasi, sehingga field ini boleh dihilangkan pada kebanyakan kasus. `url` eksplisit hanya relevan untuk grid detail, misalnya bila endpoint lookup resource tersebut tidak mengikuti pola `<resource>/lookup` standar.

### Mekanisme Kerja Select API

Saat halaman dimuat, JavaScript yang di-generate menjalankan urutan berikut:

1. Kirim `POST` ke endpoint `<apiBaseUrl>/<resource>/lookup`
2. Sertakan header `x-request-mode: static` dan body `{ "select": [...] }`
3. Tunggu response dengan format `{ "success": true, "data": [{"id": "...", "text": "..."}, ...] }`
4. Render setiap objek sebagai `<option>` dan inisialisasi sebagai Select2

### `tableField` untuk Foreign Key

Atribut `tableField` diperlukan ketika field di form (`city_id`) berbeda dengan kolom display di DataTable (`city_name`):

```
Form save/load:   city_id   → "ct-00000-0004-..."
DataTable render: city_name → "Bandung"
```

Tanpa `tableField`, DataTable mencari `city_id` di response yang berisi UUID, bukan nama kota.

## Atribut Lookup Tambahan (`autofill`, `optionColumns`, `lookupDisplay`)

Field `select` ber-`dataSource.type: "api"` mendukung empat atribut tambahan
berikut: `lookupMode`, `lookupDisplay`, `optionColumns`, dan `autofill`.
Keempatnya berlaku baik di halaman master (`pages[].fields[]`) maupun di grid
detail master-detail (`details[].fields[]`, lihat
[`master-detail.md`](./master-detail.md)) — tidak berlaku untuk
`dataFilters[]`. Keempatnya menutup kebutuhan umum form transaksi: mengisi
kolom form lain otomatis begitu opsi dipilih, menampilkan lebih dari satu
kolom informasi di dropdown, atau mengganti dropdown dengan modal pencarian
untuk master berkolom banyak. Halaman master punya sejumlah batasan tambahan
dibanding grid detail — lihat [Batasan di Halaman Master](#batasan-di-halaman-master)
di bagian bawah section ini.

Contoh deklarasi di grid detail:

```json
{
    "name": "product_id",
    "label": "Product",
    "type": "select",
    "dataSource": {
        "type": "api",
        "resource": "product",
        "url": "product/lookup",
        "select": ["product_id", "product_name", "price"],
        "lookupMode": "static",
        "lookupDisplay": "modal",
        "optionColumns": [
            { "column": "product_name" },
            { "column": "price", "format": "number", "align": "right" }
        ],
        "autofill": { "price": "price" }
    }
}
```

Contoh deklarasi setara di halaman master, pada field FK biasa (mis.
`customer_id` di header transaksi seperti `sales-order`):

```json
{
    "name": "customer_id",
    "label": "Customer",
    "type": "select",
    "required": true,
    "dataSource": {
        "type": "api",
        "resource": "customer",
        "select": ["customer_id", "customer_name", "phone"],
        "lookupDisplay": "modal",
        "optionColumns": [
            { "column": "customer_name" },
            { "column": "phone" }
        ],
        "autofill": { "customer_phone": "phone" }
    }
}
```

Pembagian peran antar atribut:

| Atribut | Peran |
|---------|-------|
| `select` | Kolom yang di-request ke endpoint lookup |
| `lookupMode` | Strategi pemuatan opsi: `"static"` (muat sekali saat form dibuka) atau `"dynamic"` (pencarian AJAX saat mengetik). Default `"static"` |
| `lookupDisplay` | Bentuk tampilan lookup: `"select2"` atau `"modal"`. Default `"select2"` |
| `optionColumns` | Kolom yang dirender, baik di dropdown maupun di tabel modal |
| `autofill` | Kolom yang disalin ke field lain di baris yang sama saat opsi dipilih |

Fitur ini bertumpu pada endpoint lookup yang membawa kolom mentah lewat
properti `row` saat `select` diminta — lihat
[`api-spec/endpoint-lookup.md`](../../api-spec/endpoint-lookup.md#kolom-row).
Generator sudah menangani ini secara otomatis; tidak ada konfigurasi tambahan
yang diperlukan di sisi backend.

### `lookupMode`

Atribut lama yang selama ini sudah dibaca generator, kini didokumentasikan
resmi di catalog ini:

| Nilai | Perilaku |
|-------|----------|
| `"static"` (default bila tidak diisi) | Seluruh opsi dimuat sekali lewat `POST` saat form dibuka, sama seperti [Mekanisme Kerja Select API](#mekanisme-kerja-select-api) di atas |
| `"dynamic"` | Opsi dimuat bertahap lewat pencarian AJAX (`GET`) saat user mengetik, cocok untuk master berjumlah besar |

`lookupMode` independen dari `lookupDisplay` — kombinasi apa pun antara
keduanya berlaku (mis. `lookupMode: "dynamic"` dengan `lookupDisplay: "modal"`
tetap valid; pencarian modal meneruskan kata kunci ke endpoint, bukan
menyaring data yang sudah dimuat).

Ketentuan di atas berlaku penuh untuk grid detail. Di halaman master,
`lookupMode: "dynamic"` BELUM tersedia — opsi field master selalu dimuat
sekali secara static saat form dibuka, terlepas dari nilai `lookupMode` yang
dideklarasikan; atribut ini tidak berefek pada field `pages[].fields[]`.

### `lookupDisplay`

Menentukan bentuk lookup pada field bersangkutan — sel grid detail atau field
form master:

| Nilai | Bentuk | Cocok untuk |
|-------|--------|-------------|
| `"select2"` (default bila tidak diisi) | Dropdown Select2 langsung di dalam sel grid atau di form | Master pendek, satu sampai dua kolom informasi |
| `"modal"` | Field menjadi input readonly plus tombol cari; pemilihan lewat modal berisi kotak pencarian dan tabel | Master panjang, kolom informasi banyak, atau butuh ruang baca lebih lapang |

Pada mode `"modal"`, kolom tabel modal diturunkan dari `optionColumns[]` yang
sama dengan mode dropdown — satu deklarasi melayani kedua bentuk. Pencarian di
modal menyaring data yang sudah dimuat bila `lookupMode: "static"` (satu-
satunya pilihan yang tersedia di halaman master saat ini), atau diteruskan
sebagai parameter pencarian ke endpoint bila `lookupMode: "dynamic"` (grid
detail saja). `autofill` (lihat di bawah) berlaku sama persis di kedua bentuk,
karena keduanya memakai jalur pembaruan state yang sama — baris grid detail
maupun form master.

Nilai selain `"select2"`/`"modal"` menghasilkan warning saat validate, lalu
fallback ke `"select2"`.

### `optionColumns`

Menentukan kolom yang dirender di dropdown maupun tabel modal — tanpa atribut
ini, hanya kolom `text` (label utama) yang tampil, persis perilaku dropdown
satu kolom sebelumnya.

```json
"optionColumns": [
    { "column": "product_name" },
    { "column": "price", "format": "number", "align": "right" }
]
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `column` | string | ✓ | Nama kolom, harus ada di `dataSource.select` (warning saat validate bila tidak) |
| `format` | string | ✗ | `"number"` memformat nilai kolom sebagai angka rata kanan. Nilai lain ditampilkan apa adanya |
| `align` | string | ✗ | Perataan teks kolom, mis. `"right"` |

Entri pertama adalah kolom utama: dipakai sebagai label yang tampil setelah
opsi terpilih (baik pada select2 maupun pada sel readonly mode modal). Jumlah
entri tidak dibatasi dua — grid detail dengan kebutuhan tampilan lebih dari
dua kolom informasi tetap didukung. Header setiap kolom diturunkan secara
otomatis dari nama kolom (Title Case, underscore menjadi spasi — mis.
`product_name` menjadi "Product Name"), tanpa atribut label terpisah.

`optionColumns` hanya berlaku untuk field `select` ber-`dataSource.type: "api"`,
baik di `pages[].fields[]` (halaman master) maupun di `details[].fields[]`
(grid detail). Dideklarasikan pada field bukan `select`, atau pada
`dataSource.type: "static"`, menghasilkan warning saat validate dan diabaikan
saat generate.

### `autofill`

Mengisi field lain begitu opsi lookup dipilih atau diganti, memakai kolom
hasil lookup sebagai sumber nilai. Pada grid detail, field tujuan berada di
baris yang sama; pada halaman master, field tujuan berada di form yang sama.

```json
"autofill": { "price": "price" }
```

Bentuknya map `"<nama field tujuan>": "<nama kolom hasil lookup>"`, sehingga
kebutuhan lebih dari satu field tujuan (mis. `uom`, `discount_percent`)
tertampung tanpa mengubah bentuk deklarasi.

| Situasi | Perilaku |
|---------|----------|
| Opsi dipilih pada entri baru (baris baru grid detail, atau record baru di form master) | Field tujuan terisi dari kolom lookup |
| Opsi diganti pada entri yang sudah terisi | Nilai lama ditimpa nilai baru |
| Entri dimuat dari data existing (mode edit) | Field tujuan **tidak** diisi ulang — nilai tersimpan (mis. harga transaksi) tetap dipertahankan, tidak tertimpa nilai master hari ini |
| Setelah terisi | Field tujuan tetap dapat diedit user secara manual (mis. diskon, harga nego) |
| Field tujuan ber-`readonly` atau ber-`calculated` | Diabaikan dari autofill, disertai warning saat validate — nilainya sudah ditentukan formula, autofill tidak boleh menimpanya |
| Field tujuan bertipe `select` | Nilai terisi seperti tipe lain, dan tampilan dropdownnya ikut disegarkan mengikuti nilai baru |

Kolom sumber (nilai pada map) sebaiknya ikut dicantumkan di `dataSource.select`
— bila tidak, validate menerbitkan warning karena kolom tersebut tidak pernah
diminta ke endpoint lookup sehingga autofill tidak akan mendapat nilai apa pun.
Nilai numerik yang datang sebagai string dari database (mis. kolom `numeric`
PostgreSQL) diparse secara otomatis saat field tujuan bertipe `number`.

Autofill murni mengisi nilai awal di form — nilai yang benar-benar tersimpan
tetap yang dikirim form saat submit, sehingga field tujuan boleh diedit ulang
kapan pun sebelum baris disimpan.

### Batasan di Halaman Master

Empat atribut di atas berlaku di halaman master dengan sejumlah batasan yang
tidak berlaku di grid detail:

- `lookupMode: "dynamic"` BELUM tersedia untuk field halaman master — opsi
  selalu dimuat sekali secara static saat form dibuka; atribut `lookupMode`
  tidak berefek di konteks ini (lihat [`lookupMode`](#lookupmode) di atas).
- Kombinasi cascade `dependsOn` dengan `autofill`/`optionColumns` pada field
  yang sama belum didukung: validate menerbitkan warning dan fitur
  bersangkutan (`autofill`/`optionColumns`) tidak dibangkitkan untuk field
  itu, sedangkan cascade-nya sendiri tetap berjalan secara normal.
- `lookupDisplay: "modal"` yang dikombinasikan dengan `dependsOn` pada field
  yang sama jatuh kembali ke dropdown cascade biasa (bukan modal), disertai
  warning saat validate.
- Field ber-`lookupDisplay: "modal"` yang menjadi induk cascade (field lain
  ber-`dependsOn` merujuk field itu) tetap didukung sepenuhnya: memilih opsi
  lewat modal juga memuat ulang opsi field anak cascade seperti biasa.

## `dataSource` di `dataFilters[]`

Filter di toolbar (`features.enableDataFilter: true`) memakai struktur `dataSource` yang sama dengan field `select`:

```json
{
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
}
```

Lihat [`features.md`](./features.md) untuk detail `enableDataFilter`.

## Perbandingan Static vs API

| Aspek | Static | API |
|-------|--------|-----|
| Sumber data | Didefinisikan di payload | Dimuat dari endpoint `/<resource>/lookup` |
| Waktu load | Tersedia langsung saat halaman dibuka | Memerlukan HTTP POST saat halaman dimuat |
| Cocok untuk | Data tetap, jumlah sedikit (<10 opsi) | Data dinamis, jumlah banyak, sering berubah |
| Dependensi | Tidak memerlukan backend | Memerlukan backend yang berjalan |
| `tableField` | Tidak diperlukan | Diperlukan jika display value berbeda dari foreign key |

## Aturan Validasi

| Aturan | Hasil |
|--------|-------|
| Field `select` tanpa `dataSource` | Warning: `"Field '<name>' is of type 'select' but has no dataSource"` |
| `dataSource.type` selain `"static"` / `"api"` | Tidak divalidasi formal oleh validator umum, tetapi plugin runtime mengabaikan dataSource yang tidak dikenali |

Untuk field `select` ber-`dataSource.type: "api"`, baik di `pages[].fields[]`
(halaman master) maupun di `details[].fields[]` (grid detail master-detail),
berlaku aturan tambahan berikut untuk `autofill`, `optionColumns`, dan
`lookupDisplay`:

| Aturan | Hasil |
|--------|-------|
| `dataSource.autofill` merujuk kolom sumber yang tidak ada di `dataSource.select` | Warning; autofill tetap digenerate |
| `dataSource.autofill` menunjuk field tujuan yang tidak ada di `fields` (halaman atau detail) yang sama | Warning; pasangan itu diabaikan dari codegen autofill |
| `dataSource.autofill` menunjuk field tujuan ber-`readonly` atau ber-`calculated` | Warning; pasangan itu diabaikan dari codegen autofill |
| `dataSource.optionColumns` dideklarasikan pada field bukan `select`, atau `dataSource.type` bukan `"api"` | Warning |
| `dataSource.optionColumns[].column` tidak ada di `dataSource.select` | Warning |
| `dataSource.lookupDisplay` bernilai selain `"select2"` / `"modal"` | Warning; fallback ke `"select2"` |

Khusus field halaman master, berlaku tambahan berikut (lihat
[Batasan di Halaman Master](#batasan-di-halaman-master) di atas):

| Aturan | Hasil |
|--------|-------|
| Field halaman master ber-`dependsOn` yang juga mendeklarasikan `autofill`/`optionColumns` pada field yang sama | Warning; `autofill`/`optionColumns` tidak dibangkitkan, cascade tetap berjalan |
| Field halaman master ber-`dependsOn` dengan `lookupDisplay: "modal"` pada field yang sama | Warning; jatuh kembali ke dropdown cascade biasa |

Seluruh aturan tambahan di atas berupa warning, bukan error — generate tetap berjalan.

> **Catatan:** Validator UDF tidak memvalidasi struktur internal `dataSource` (mis. apakah `options[].value` ada). Validasi tingkat plugin atau runtime menangani sisanya. Jika ada plugin custom yang menambah tipe `dataSource` baru, dokumentasi plugin tersebut yang menjadi referensi.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Daftar provinsi Indonesia (34 opsi) | `static` karena data jarang berubah |
| Daftar kota di Indonesia (500+ opsi) | `api` karena data dinamis dan banyak |
| Status pesanan (4 opsi: Draft, Submitted, Approved, Rejected) | `static` |
| Daftar customer (ratusan, bisa bertambah) | `api` |
| Daftar tipe transaksi (5 opsi tetap) | `static` |

---

← [`field-attributes.md`](./field-attributes.md) | [Selanjutnya: `id-generation.md`](./id-generation.md) →
