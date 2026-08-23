# Tipe Field — `fields[].type`

> Daftar resmi tipe field yang dikenali validator UDF, beserta library frontend yang dipakai per tipe.

## Daftar Tipe Valid

Tipe field yang dikenali oleh validator (source code: `validator/constants.rs::VALID_FIELD_TYPES`):

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
| `maxlength` | integer | Batas maksimal karakter. Browser memblokir input melebihi batas |
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
| `min` | number | Nilai minimum yang diizinkan |
| `max` | number | Nilai maksimum yang diizinkan |
| `step` | number | Increment/decrement per klik spinner |

Nilai yang dikirim ke backend dikonversi dengan `parseFloat()` di JavaScript yang di-generate.

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
| Tabel list | Badge berwarna (hijau untuk Active, abu-abu untuk Inactive) |
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

Format yang dipakai generator:

| Aspek | Nilai |
|-------|-------|
| Format simpan | `Y-m-d` (contoh: `2025-03-28`) |
| Format tampilan | `d/m/Y` (contoh: `28/03/2025`) |

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
| Format simpan | `Y-m-d H:i` (contoh: `2025-03-28 14:30`) |
| Format tampilan | `d/m/Y H:i` (contoh: `28/03/2025 14:30`) |

Sering dikombinasikan dengan `readonly: true` untuk field yang diisi backend (`created_at`, `updated_at`).

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
| `dateFormat` | `Y-m-d` | `Y-m-d H:i` | `H:i` |
| `altInput` | `true` | `true` | — |
| `altFormat` | `d/m/Y` | `d/m/Y H:i` | — |
| `enableTime` | — | `true` | `true` |
| `noCalendar` | — | — | `true` |
| `time_24hr` | — | `true` | `true` |
| `allowInput` | `true` | `true` | `true` |

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
