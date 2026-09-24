# Atribut Field — Common dan Per Tipe

> Daftar atribut yang dikenali validator di setiap entry `fields[]`.

## Atribut Wajib

Setiap field minimal harus memiliki tiga atribut berikut. Tanpa salah satu, validator menghasilkan error.

| Atribut | Tipe | Error jika Tidak Ada |
|---------|------|---------------------|
| `name` | string | `"Field does not have a 'name' property"` |
| `label` | string | `"Field '<name>' does not have a 'label' property"` |
| `type` | string | `"Field '<name>' does not have a 'type' property"` |

```json
{
    "name": "contact_name",
    "label": "Nama Kontak",
    "type": "text"
}
```

Format `name`: snake_case direkomendasikan. Generator mengonversi secara otomatis ke `kebab-case` untuk HTML `id` dan `camelCase` untuk variabel JavaScript.

## Atribut Umum (Berlaku untuk Semua Tipe)

| Atribut | Tipe | Default | Keterangan |
|---------|------|---------|-----------|
| `required` | boolean | `false` | Validasi wajib diisi. Label ditambah asterisk merah |
| `inTable` | boolean | `false` | Tampil sebagai kolom di tabel list |
| `tableOrder` | integer | — | Urutan kolom di tabel (mulai dari 1). Diperlukan jika `inTable: true` |
| `tableField` | string | — | Nama kolom alternatif di response DataTable. Untuk foreign key |
| `width` | string | — | Lebar kolom DataTable (`"200px"`, `"30%"`). Default: auto |
| `placeholder` | string | — | Teks petunjuk di dalam input |
| `readonly` | boolean | `false` | Field hanya baca, tidak divalidasi, tidak disertakan saat save |
| `maxlength` | integer | — | Batas maksimal karakter. Berlaku untuk `text` dan `textarea` |
| `defaultValue` | any | — | Nilai default saat mode Add. Tipe data menyesuaikan field type |
| `editorMode` | string | — | Mode editor khusus. Valid: `"hidden"`, `"readonly"` |

### `editorMode`

Field `editorMode` mengontrol bagaimana field ditampilkan di form:

| Nilai | Behavior |
|-------|----------|
| `"hidden"` | Field dirender sesuai tipenya tetapi tidak ditampilkan. Nilainya diisi `defaultValue` saat Add dan diisi dari record saat Edit, lalu dikirim seperti field biasa |
| `"readonly"` | Field dirender tetapi tidak bisa diedit. Setara dengan `readonly: true` |

Field `hidden` berperilaku persis seperti field biasa, hanya tanpa tampilan di layar: elemen form tetap ada (termasuk daftar opsi untuk `select`), tetapi pembungkusnya disembunyikan. Pada mode Add, elemen diisi `defaultValue` (nilai statis, `today`/`now`, atau IdGen) lalu dibaca saat simpan. Pada mode Edit, elemen diisi dari record lalu dikirim kembali apa adanya, sehingga nilai yang tersimpan tidak tertimpa. `defaultValue` opsional; tanpa `defaultValue`, field kosong saat Add dan mengikuti record saat Edit. Field `hidden` tidak divalidasi di sisi client dan tidak perlu disebut di `fieldRows` (lihat [`field-rows.md`](./field-rows.md)).

Mode ini cocok untuk field yang nilainya harus mengikuti record tetapi tidak boleh diubah lewat form, misalnya status workflow, kunci teknis, atau kolom agregat yang dihitung ulang backend.

Valid editor mode yang dikenali: `"hidden"`, `"readonly"`. Nilai lain menghasilkan error.

### `inTable` dan `tableOrder`

Jika `inTable: true` tanpa `tableOrder`, validator menghasilkan warning dan urutan kolom menjadi tidak terprediksi:

```
Field '<name>' has inTable:true but no tableOrder is defined
```

### `tableField` untuk Foreign Key

Atribut `tableField` diperlukan ketika nama field di form (foreign key) berbeda dengan nama kolom di response DataTable (display value dari JOIN di backend).

```
Form (save/load):   city_id   → "ct-00000-0004-..."
DataTable (render): city_name → "Bandung"
```

Tanpa `tableField`, DataTable mencari kolom `city_id` yang hanya berisi UUID, bukan nama kota.

## Atribut Khusus Per Tipe

Detail atribut khusus untuk setiap tipe ada di [`field-types.md`](./field-types.md). Ringkasan:

| Tipe | Atribut Khusus |
|------|---------------|
| `text` | `maxlength`, `placeholder` |
| `textarea` | `rows`, `maxlength`, `placeholder` |
| `number` | `min`, `max`, `step`, `format`, `decimalPlaces` |
| `checkbox` | `defaultValue`, `checkboxText.checked`, `checkboxText.unchecked` |
| `select` | `dataSource`, `tableField` |
| `date` / `timestamp` | `dateFormat`, `temporalType` (khusus `timestamp`) |
| `time` | — (Flatpickr config otomatis) |

### `format` dan `decimalPlaces` untuk Field `number`

Field `number` di halaman master mendukung dua atribut tambahan untuk memformat tampilan angka, baik di form input maupun di kolom DataTable:

```json
{
    "name": "selling_price",
    "label": "Selling Price",
    "type": "number",
    "format": "currency",
    "decimalPlaces": 2
}
```

| Atribut | Tipe | Keterangan |
|---------|------|-----------|
| `format` | string | Format tampilan angka. Valid: `"number"` (pemisah ribuan biasa) atau `"currency"` (format mata uang) |
| `decimalPlaces` | integer | Jumlah digit desimal yang ditampilkan. Default `0` bila tidak diisi |

Kedua atribut berlaku pada form input dan kolom DataTable halaman master dengan nilai yang sama, sehingga satu deklarasi di field menghasilkan tampilan yang seragam di keduanya. Pada form input, nilai yang dimuat saat edit ditampilkan dengan format ini, input diformat otomatis saat diketik, dan diparse balik ke angka murni saat disimpan. Karakter pemisah ribuan dan desimal mengikuti `appConfig.numberFormat.locale` (lihat [`app-config.md`](./app-config.md#properti)).

Jumlah digit desimal tidak diatur di level aplikasi. Nilai `decimalPlaces` berasal dari skala kolom database: `payload generate` menulis skala itu sebagai `scale` di RDF, lalu `payload migrate` menurunkannya ke `decimalPlaces` field UDF. RDF lama yang skalanya masih tersimpan di `precision` tetap terbaca sebagai fallback, disertai warning migrate. Field tanpa `decimalPlaces` ditampilkan tanpa digit desimal.

Nilai `format` juga turunan RDF: `payload migrate` menulis `"currency"` hanya bila RDF mendeklarasikan `constraints.format: "currency"` pada field numerik tersebut, dan `"number"` untuk field lain apa pun namanya (`price`, `harga`, `nilai_harga`). Migrate ulang dengan `--overwrite` selalu menimpa `format` dari RDF, tidak mempertahankan nilai tulisan tangan sebelumnya. Field tanpa key `format` sama sekali (misalnya UDF yang ditulis manual, bukan hasil migrate) ditampilkan tanpa pemformatan, angka mentah apa adanya; tulis `format` dan `decimalPlaces` secara eksplisit pada UDF tulisan tangan agar tampilannya terformat.

Untuk field `number` di grid detail master-detail, atribut yang sama berlaku juga pada input, kolom tabel, dan baris ringkasan (`summary`), lihat [Format Tampilan Angka](./master-detail.md#format-tampilan-angka-format-dan-decimalplaces) dan [Properti `summary`](./master-detail.md#properti-summary).

### Atribut Field `date` dan `timestamp`

| Atribut | Tipe | Keterangan |
|---------|------|-----------|
| `dateFormat` | string | Pola tampilan bagian tanggal untuk field ini, menggantikan `appConfig.dateFormat`. Nilai yang didukung: `yyyy-MM-dd`, `dd/MM/yyyy`, `dd-MM-yyyy`, `MM/dd/yyyy`, `yyyy/MM/dd`. Penulisan huruf kecil lama (`dd/mm/yyyy`) tetap diterima |
| `temporalType` | string | Khusus `timestamp`: `timestamp` (default) atau `timestamptz`. Diisi `payload migrate` untuk kolom `timestamptz`. Lihat [`field-types.md`](./field-types.md#timestamp) |

`dateFormat` hanya mengubah tampilan. Frontend selalu membaca nilai dari backend dengan pola `appConfig.dateFormat` dan `appConfig.dateTimeFormat`, karena backend memakai satu pola untuk semua field (lihat [`rdf/datetime-fields.md`](../rdf/datetime-fields.md#satu-standar-per-aplikasi)).

Validator menolak nilai di luar daftar di atas. `temporalType` pada field selain `timestamp` juga ditolak. Atribut `valueFormat` tidak lagi didukung dan ditolak validator; hapus atribut itu lalu jalankan ulang `payload migrate`.

## `defaultValue` Khusus

Atribut `defaultValue` dapat berupa nilai literal atau object dengan `source: "idgen"` untuk auto-generation. Lihat [`id-generation.md`](./id-generation.md) untuk detail pola idgen.

### Literal Default

```json
{"name": "is_active", "type": "checkbox", "defaultValue": true}
{"name": "qty", "type": "number", "defaultValue": 1}
{"name": "remark", "type": "text", "defaultValue": "N/A"}
```

### IdGen Default

```json
{
    "name": "invoice_no",
    "type": "text",
    "defaultValue": {
        "source": "idgen",
        "mode": "number",
        "resource": "invoice",
        "format": "yyyymm",
        "numDigits": 5,
        "separator": "-",
        "reserve": true,
        "ttl": 60
    }
}
```

Pola idgen punya batasan tipe field: hanya `number`, `text`, `textarea`. Tipe lain menghasilkan error.

## Cakupan `defaultValue` dan `width` di Grid Detail Master-Detail

Pada `details[].fields[]` (grid detail master-detail, lihat
[`master-detail.md`](./master-detail.md)), kedua atribut berikut punya
perilaku tambahan yang tidak berlaku di halaman master:

- `defaultValue` pada field `number`: dipakai sebagai nilai awal baris baru
  yang ditambahkan ke grid, bukan hanya nilai default form. Field `checkbox`
  dan opsi pertama `select` static tetap mengikuti perilaku umum yang sudah
  dijelaskan di atas.
- `width`: mengatur lebar kolom grid detail, dengan default yang berbeda dari
  lebar kolom DataTable halaman master (baris tabel di atas).

Detail lengkap kedua perilaku ada di
[Nilai Awal Baris Baru](./master-detail.md#nilai-awal-baris-baru-defaultvalue)
dan [Lebar Kolom Grid](./master-detail.md#lebar-kolom-grid-width) pada
`master-detail.md`.

## Validasi Cross-Field

### Reservasi Unik per Page

Maksimal 1 field per page yang boleh memakai `defaultValue.reserve: true`. Lebih dari 1 menghasilkan error:

```
Only one field per page may use defaultValue.reserve=true. Found: <name1>, <name2>
```

### Konsistensi `fields` dan `fieldRows`

Jika `fieldRows` mereferensikan nama field yang tidak ada di `fields`, validator menghasilkan warning:

```
fieldRows references fields that are not defined in 'fields': <name>
```

Sebaliknya, field yang ada di `fields` tetapi tidak direferensikan di `fieldRows` juga menghasilkan warning karena tidak akan muncul di form. Field `editorMode: "hidden"` dikecualikan: field ini tetap dirender (tersembunyi) setelah seluruh baris grid meski tidak disebut di `fieldRows`.

```
The following fields are defined in 'fields' but are not referenced in 'fieldRows' and
will not be rendered in the form: <names>. Add them to 'fieldRows' to make them visible,
or set editorMode:'hidden' if they are intentionally excluded from the form.
```

## Konvensi Penamaan

Detail lengkap di [`naming-convention.md`](./naming-convention.md). Ringkasan:

| Format Asal | Format Tujuan | Contoh | Digunakan Untuk |
|-------------|---------------|--------|-----------------|
| `snake_case` | `kebab-case` | `contact_name` → `contact-name` | HTML `id` dan CSS class |
| `snake_case` | `camelCase` | `contact_name` → `contactName` | Variabel JavaScript |

---

← [`field-types.md`](./field-types.md) | [Selanjutnya: `data-source.md`](./data-source.md) →
