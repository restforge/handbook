# Tipe Field — `fields[].type`

> Daftar resmi tipe field yang dikenali validator UDF, beserta library frontend yang dipakai per tipe.

## Daftar Tipe Valid

Tipe field yang dikenali validator:

```
checkbox, date, number, select, text, textarea, time, timestamp
```

| Tipe | HTML Element | Library Otomatis | Atribut Khusus |
|------|-------------|------------------|----------------|
| `text` | `<input type="text">` | — | `maxlength`, `placeholder` |
| `textarea` | `<textarea>` | — | `rows`, `maxlength`, `placeholder` |
| `number` | `<input type="number">` | — | `min`, `max`, `step` |
| `checkbox` | toggle switch | — | `defaultValue`, `checkboxText` |
| `select` | `<select>` | Select2 | `dataSource`, `tableField` |
| `date` | `<input type="text">` | Flatpickr | — |
| `timestamp` | `<input type="text">` | Flatpickr | — |
| `time` | `<input type="text">` | Flatpickr | — |

Jika `type` tidak ada di daftar valid, validator menghasilkan error:

```
Field '<name>' has an invalid type: '<value>'. Valid types are:
checkbox, date, number, select, text, textarea, time, timestamp
```

## `text`

Input teks satu baris.

```json
{
    "name": "contact_name",
    "label": "Nama Kontak",
    "type": "text",
    "maxlength": 255,
    "placeholder": "Masukkan nama"
}
```

| Atribut Khusus | Tipe | Keterangan |
|----------------|------|-----------|
| `maxlength` | integer | Batas maksimal karakter. Browser memblokir input melebihi batas, dan form menolak nilai yang lebih panjang sebelum request dikirim |
| `placeholder` | string | Teks petunjuk saat input kosong |

## `textarea`

Input teks multi-baris.

```json
{
    "name": "address",
    "label": "Alamat",
    "type": "textarea",
    "rows": 3,
    "maxlength": 500
}
```

| Atribut Khusus | Tipe | Default | Keterangan |
|----------------|------|---------|-----------|
| `rows` | integer | `4` | Jumlah baris awal yang terlihat |
| `maxlength` | integer | — | Batas maksimal karakter |
| `placeholder` | string | — | Teks petunjuk |

**Dihasilkan secara otomatis dari tipe `text`**: saat `migrate payload`, field RDF/SDF bertipe `text` dipetakan ke UDF `textarea` (resolver Rule 8.5). Karena `text` bersifat unbounded, textarea hasilnya **tidak** memiliki `maxlength` kecuali ada `constraints.maxLength`, dan `rows` diatur secara eksplisit ke `3`. Lihat [`sdf/field-types.md`](../sdf/field-types.md) dan [`rdf/field-validation.md`](../rdf/field-validation.md) untuk rantai pemetaan `text` → `textarea`.

## `number`

Input angka dengan kontrol spinner.

```json
{
    "name": "stock_qty",
    "label": "Jumlah Stok",
    "type": "number",
    "min": 0,
    "max": 100,
    "step": 1
}
```

| Atribut Khusus | Tipe | Keterangan |
|----------------|------|-----------|
| `min` | number | Nilai minimum yang diizinkan. Nilai di bawahnya ditolak sebelum request dikirim, dengan pesan di bawah field. Bila bernilai 0 atau lebih, form menolak tanda minus saat pengguna mengetik |
| `max` | number | Nilai maksimum yang diizinkan. Nilai di atasnya ditolak sebelum request dikirim |
| `step` | number | Increment/decrement per klik spinner |

Angka 0 tetap tampil dan dikirim ke backend sebagai 0. Field yang dikosongkan dikirim sebagai `null`, bukan 0. Field wajib (`required`) yang diisi 0 dianggap terisi, sedangkan field wajib yang kosong ditolak sebelum request dikirim. Satu tanda minus di depan angka diterima, kecuali pada field dengan `min` bernilai 0 atau lebih.

## `checkbox`

Boolean yang di-render sebagai toggle switch (bukan native checkbox HTML).

```json
{
    "name": "is_active",
    "label": "Status",
    "type": "checkbox",
    "defaultValue": true,
    "checkboxText": {
        "checked": "Active",
        "unchecked": "Inactive"
    }
}
```

| Atribut Khusus | Tipe | Default | Keterangan |
|----------------|------|---------|-----------|
| `defaultValue` | boolean | `false` | Nilai awal saat mode Add |
| `checkboxText` | object | — | Label dinamis untuk state checked/unchecked |
| `checkboxText.checked` | string | — | Label saat toggle ON |
| `checkboxText.unchecked` | string | — | Label saat toggle OFF |

Behavior tampilan:

| Konteks | Behavior |
|---------|----------|
| Tabel list | Badge berwarna. Dengan `statusBadge: true`, tampil sebagai kolom Status berpalet tetap (lihat [Kolom Status](#kolom-status-statusbadge)) |
| Form mode Add | Toggle ON jika `defaultValue: true`. Label menampilkan `checkboxText.checked` |
| Form mode Edit | Toggle mengikuti nilai dari data. Label berubah dinamis sesuai state |

## `select`

Dropdown dengan library Select2. Wajib disertai `dataSource` (lihat [`data-source.md`](./data-source.md)).

```json
{
    "name": "city_id",
    "label": "Kota",
    "type": "select",
    "tableField": "city_name",
    "dataSource": {
        "type": "api",
        "resource": "city",
        "select": ["city_id", "city_name"]
    }
}
```

| Atribut Khusus | Tipe | Keterangan |
|----------------|------|-----------|
| `dataSource` | object | Sumber data dropdown (wajib). Lihat [`data-source.md`](./data-source.md) |
| `tableField` | string | Nama kolom alternatif di response DataTable, untuk field foreign key |
| `dependsOn` | string | Nama (`name`) field `select` lain di page yang sama yang menjadi induk cascade. Lihat [Select Cascade (`dependsOn`)](#select-cascade-dependson) |

Jika field `select` tidak memiliki `dataSource`, validator menghasilkan warning dan dropdown di-render kosong:

```
Field '<name>' is of type 'select' but has no dataSource
```

### Select Cascade (`dependsOn`)

Default-nya, setiap field `select` dengan `dataSource.type: "api"` berdiri sendiri — opsi dropdown dimuat penuh sekali saat form dibuka, tidak bergantung field lain. Untuk field berjenjang (mis. Provinsi → Kabupaten/Kota → Kecamatan), tambahkan `dependsOn` yang menunjuk ke `name` field `select` induk di page yang sama:

```json
{
    "name": "province_id",
    "label": "Provinsi",
    "type": "select",
    "dataSource": { "type": "api", "resource": "province", "select": ["province_id", "province_name"] }
},
{
    "name": "city_id",
    "label": "Kabupaten/Kota",
    "type": "select",
    "dependsOn": "province_id",
    "dataSource": { "type": "api", "resource": "city", "select": ["city_id", "city_name"] }
},
{
    "name": "district_id",
    "label": "Kecamatan",
    "type": "select",
    "dependsOn": "city_id",
    "dataSource": { "type": "api", "resource": "district", "select": ["district_id", "district_name"] }
}
```

Behavior:

| Aspek | Behavior |
|-------|----------|
| Saat form dibuka | Field dengan `dependsOn` TIDAK ikut memuat opsi; dropdown-nya cuma placeholder sampai induknya dipilih |
| Saat field induk dipilih/berubah | Opsi field anak di-fetch ulang via endpoint `/lookup` dengan `where: [{"key": "<dependsOn>", "value": "<nilai induk>"}]` — nama field induk menjadi key, konvensi sama dengan filter cascade |
| Saat field induk dikosongkan | Opsi field anak kembali ke placeholder saja (tidak fetch); nilai field anak ikut dikosongkan |
| Cascade berjenjang | Mengubah field di tengah rantai otomatis mengosongkan nilai DAN opsi seluruh field di bawahnya, tidak cuma anak langsung |
| Mode Edit/View/Duplicate | Rantai ter-resolve otomatis berurutan dari akar saat record dimuat: nilai field induk diisi, opsi field anak di-fetch berdasarkan nilai itu, lalu nilai field anak diisi — berlanjut sampai field paling bawah |
| Nilai tersimpan yang sudah tidak ada di opsi hasil fetch (mis. data nonaktif) | Tetap ditampilkan sebagai opsi tambahan, konsisten dengan behavior field `select` biasa |
| Cakupan tipe | Hanya berlaku untuk `dataSource.type: "api"`. Diabaikan (tanpa error) untuk `dataSource.type: "static"` |
| `defaultValue` | Diabaikan pada field ber-`dependsOn` — nilai bawaan tidak bisa divalidasi terhadap induk yang belum dipilih |

`dependsOn` wajib menunjuk ke `name` field bertipe `select` lain di `fields[]` page yang sama. Validator memunculkan warning (bukan error, form tetap dibuat) untuk kondisi berikut:

- target `dependsOn` tidak ditemukan di page yang sama, atau menunjuk field itu sendiri;
- target `dependsOn` bukan field bertipe `select`;
- field ber-`dependsOn` tidak memakai `dataSource.type: "api"` (mis. `static` atau tanpa `dataSource`);
- field ber-`dependsOn` juga memakai `defaultValue`;
- rantai `dependsOn` membentuk siklus (mis. field A bergantung field B, field B bergantung field A).

Rantai `dependsOn` yang siklik adalah konfigurasi invalid; behavior-nya saat runtime tidak dijamin (fail-silent), sama seperti filter cascade.

Lihat juga [Filter Cascade (`dependsOn`)](./features.md#filter-cascade-dependson) untuk mekanisme serupa pada `dataFilters[]`, dan [`data-source.md`](./data-source.md) untuk detail `dataSource.type: "api"`.

## `date`

Input tanggal saja, di-render dengan Flatpickr.

```json
{
    "name": "birth_date",
    "label": "Tanggal Lahir",
    "type": "date"
}
```

| Aspek | Nilai |
|-------|-------|
| Format simpan | `yyyy-MM-dd` (contoh: `2025-03-28`) |
| Format tampilan | `appConfig.dateFormat` (default `yyyy-MM-dd`), atau `dateFormat` field bila diisi |

Nilai `date` adalah tanggal kalender. Tanggal yang tampil tidak bergeser meskipun zona waktu komputer pembaca berbeda.

## `timestamp`

Input tanggal + waktu, di-render dengan Flatpickr (mode `enableTime: true`).

```json
{
    "name": "registered_at",
    "label": "Waktu Registrasi",
    "type": "timestamp",
    "readonly": true
}
```

| Aspek | Nilai |
|-------|-------|
| Format simpan | `yyyy-MM-dd HH:mm:ss` (contoh: `2025-03-28 14:30:05`), atau ISO UTC untuk `timestamptz` |
| Format tampilan | `appConfig.dateTimeFormat` (default `yyyy-MM-dd HH:mm:ss.SSS`). Bagian tanggal diganti `dateFormat` field bila diisi. Form tidak menampilkan milidetik |

Sering dikombinasikan dengan `readonly: true` untuk field yang diisi backend (`created_at`, `updated_at`).

Kolom database `timestamptz` juga memakai tipe `timestamp`, ditandai `temporalType: "timestamptz"`. `payload migrate` menulis tanda ini secara otomatis.

| `temporalType` | Arti | Tampilan | Nilai yang dikirim |
|----------------|------|----------|--------------------|
| `timestamp` (default) | Jam dinding zona aplikasi | Apa adanya, tidak terpengaruh zona komputer pembaca | `yyyy-MM-dd HH:mm:ss` |
| `timestamptz` | Momen absolut | Dikonversi ke zona komputer pembaca | ISO UTC, mis. `2025-03-28T07:30:05.000Z` |

Backend selalu mengirim nilai `timestamptz` sebagai ISO UTC, sehingga konversi ke zona komputer pembaca selalu berlaku untuk field ini.

## `time`

Input waktu saja, di-render dengan Flatpickr (mode `noCalendar: true`).

```json
{
    "name": "start_time",
    "label": "Jam Mulai",
    "type": "time"
}
```

| Aspek | Nilai |
|-------|-------|
| Format simpan | `H:i` (contoh: `14:30`) |
| Format tampilan | `H:i` (contoh: `14:30`) |

## Kolom Status (`statusBadge`)

Field yang diberi `statusBadge: true` tampil di tabel list sebagai kolom Status dengan label dan palet tetap. Palet ini sama di semua halaman dan semua plugin, sehingga status yang sama selalu tampil dengan bentuk yang sama.

```json
{
    "name": "status",
    "label": "Status",
    "type": "select",
    "inTable": true,
    "statusBadge": true,
    "dataSource": {
        "type": "static",
        "options": [
            { "value": "draft", "text": "Draft" },
            { "value": "posted", "text": "Posted" },
            { "value": "cancelled", "text": "Cancelled" }
        ]
    }
}
```

| Atribut | Tipe | Default | Keterangan |
|---------|------|---------|-----------|
| `statusBadge` | boolean | `false` | Menandai field sebagai kolom Status. Hanya berlaku pada field `checkbox` dan field `select` dengan `dataSource.type: "static"`. Satu page hanya boleh punya satu field `statusBadge` |

Palet dicocokkan dengan nilai field, bukan dengan teks label, sehingga tetap berlaku saat label diterjemahkan:

| Nilai | Tampilan |
|-------|----------|
| `active` (checkbox bernilai true) | Titik hijau dan teks, tanpa badge |
| `inactive` (checkbox bernilai false) | Badge abu |
| `draft` | Badge abu |
| `open` | Badge biru |
| `posted` | Badge hijau |
| `closed` | Badge abu gelap |
| `cancelled` | Badge merah |
| Nilai lain | Badge netral, sama seperti `draft` |

Label diambil dari `text` pada option, atau dari `checkboxText` untuk checkbox. Nilai yang tidak punya option ditampilkan dengan huruf awal kapital, misalnya `posted` menjadi `Posted`. Nilai untuk pengurutan, filter, dan export tetap nilai asli dari database.

Properti `badge` pada option menimpa palet tetap untuk option tersebut. Option `{ "value": "posted", "text": "Posted", "badge": "badge-light-info" }` tampil dengan class `badge-light-info`, sedangkan option lain tetap mengikuti palet.

Modal Change Status menampilkan `Current Status` dengan label dan warna yang sama dengan kolom Status di list. Label dan warna modal diambil dari field yang ditunjuk `workflow.statusField`.

Field `checkbox` ber-`statusBadge: true` juga menambahkan item `Deactivate` atau `Activate` ke menu Actions. Perilakunya dijelaskan di [`features.md`](./features.md#nonaktifkan-dan-aktifkan-kembali).

[`payload migrate`](../../commands/restforge-backend/payload/migrate.md#field-status-enum-menjadi-statusfilter) mengisi `statusBadge: true` secara otomatis pada field status utama. Nilai `false` melepas penandaan tersebut dan tetap dipertahankan saat migrate berikutnya. Field `checkbox` dan `select` tanpa `statusBadge` tampil seperti biasa.

## Library Otomatis Per Tipe

Generator hanya menyertakan library yang benar-benar dipakai oleh page:

| Library | Di-include Jika | CDN |
|---------|----------------|-----|
| Select2 | Ada field bertipe `select` | `cdn.jsdelivr.net/npm/select2@4.1.0-rc.0` |
| Flatpickr | Ada field bertipe `date`, `timestamp`, atau `time` | `cdn.jsdelivr.net/npm/flatpickr` |

Inisialisasi bersifat kondisional:

- `initFormSelect2()` hanya di-generate jika ada minimal 1 field `select`
- `initFormFlatpickr()` hanya di-generate jika ada minimal 1 field `date`, `timestamp`, atau `time`

## Perbandingan Konfigurasi Flatpickr

| Opsi Flatpickr | `date` | `timestamp` | `time` |
|----------------|--------|-------------|--------|
| `dateFormat` | `Y-m-d` | `Y-m-d H:i:S` | `H:i` |
| `altInput` | `true` | `true` | — |
| `altFormat` | Mengikuti pola tampilan | Mengikuti pola tampilan, tanpa milidetik | — |
| `enableTime` | — | `true` | `true` |
| `noCalendar` | — | — | `true` |
| `time_24hr` | — | `true` | `true` |
| `allowInput` | `true` | `true` | `true` |

`dateFormat` Flatpickr adalah bentuk nilai yang dikirim ke backend. Pola tampilan hanya memengaruhi `altFormat`, yaitu teks yang dilihat dan diketik pengguna. Aturan membaca dan menyimpan nilai tanggal dijelaskan di [`app-config.md`](./app-config.md#pola-tanggal-dateformat-dan-datetimeformat).

## Kapan Pakai Tipe yang Mana

| Kebutuhan | Tipe yang Dipakai |
|-----------|------------------|
| Identifier, nama, kode, email, phone | `text` |
| Alamat, deskripsi, catatan panjang | `textarea` |
| Harga, kuantitas, skor, ID numerik | `number` |
| Flag aktif/nonaktif, on/off | `checkbox` |
| Kategori, foreign key, lookup | `select` |
| Tanggal saja tanpa jam | `date` |
| Tanggal + jam (timestamp transaksi) | `timestamp` |
| Jam saja tanpa tanggal | `time` |

---

← [`page-anatomy.md`](./page-anatomy.md) | [Selanjutnya: `field-attributes.md`](./field-attributes.md) →
