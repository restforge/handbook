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

Jika field `select` tidak memiliki `dataSource`, validator menghasilkan warning dan dropdown di-render kosong:

```
Field '<name>' is of type 'select' but has no dataSource
```

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

Generator hanya meng-include library yang benar-benar dipakai oleh page:

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
