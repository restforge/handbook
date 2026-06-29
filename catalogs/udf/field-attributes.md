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
| `"hidden"` | Field tidak dirender di form tetapi tetap di-submit. Wajib disertai `defaultValue` |
| `"readonly"` | Field dirender tetapi tidak bisa diedit. Setara dengan `readonly: true` |

Jika `editorMode: "hidden"` tanpa `defaultValue`, validator menghasilkan warning:

```
Field '<name>' uses editorMode:'hidden' but has no defaultValue
```

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
| `number` | `min`, `max`, `step` |
| `checkbox` | `defaultValue`, `checkboxText.checked`, `checkboxText.unchecked` |
| `select` | `dataSource`, `tableField` |
| `date` / `timestamp` / `time` | — (Flatpickr config otomatis) |

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

Sebaliknya, field yang ada di `fields` tetapi tidak direferensikan di `fieldRows` (kecuali `editorMode: "hidden"`) juga menghasilkan warning karena tidak akan muncul di form:

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
