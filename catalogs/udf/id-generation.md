# ID Generation — `defaultValue.source: "idgen"`

> Pola `defaultValue` yang memicu generator backend untuk membuat ID otomatis (nomor invoice, kode, PIN, serial).

Pola ini mengonsumsi feature **ID Generator** di backend. Halaman ini fokus pada cara mendeklarasikan idgen pada field UDF, sedangkan service lengkapnya (HTTP API, mode, reuse pool, reservation, dan pemakaian di processor) didokumentasikan di [`features/id-generator/`](../../features/id-generator/README.md).

## Konsep Dasar

Beberapa field membutuhkan nilai yang di-generate secara otomatis di backend (mis. nomor invoice berurutan, kode unik per tahun). UDF menyediakan pola `defaultValue.source: "idgen"` untuk mendeklarasikan kebutuhan ini secara deklaratif tanpa menulis logika JavaScript.

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
        "separator": "-"
    }
}
```

## Tipe Field yang Didukung

Pola idgen hanya boleh dipakai pada tiga tipe field:

```
number, text, textarea
```

Tipe lain (mis. `checkbox`, `date`, `select`) menghasilkan error:

```
Field '<name>' cannot use IdGen defaultValue with type '<type>'.
Supported types: number, text, textarea
```

## Mode IdGen

Atribut `mode` menentukan algoritma generation. Mode yang valid:

```
code, number, pin, random, serial
```

| Mode | Tujuan | Atribut yang Diperlukan |
|------|--------|-------------------------|
| `number` | Nomor urut dengan prefix berbasis tanggal | `format`, `numDigits` |
| `pin` | PIN angka acak n-digit | `digits` (default 6) |
| `code` | Kode dengan pattern tertentu | `pattern` |
| `serial` | Serial number dengan pattern | `pattern` |
| `random` | String acak dengan pattern | `pattern` |

Jika `mode` tidak ada atau bukan salah satu di atas, validator menghasilkan error:

```
Field '<name>' defaultValue.mode is invalid: '<value>'.
Valid modes: code, number, pin, random, serial
```

## Atribut Wajib Umum

| Atribut | Tipe | Keterangan |
|---------|------|-----------|
| `source` | string | Harus bernilai `"idgen"` |
| `mode` | string | Salah satu mode di atas |
| `resource` | string | Identifier resource untuk namespace generator. Tidak boleh mengandung `:` |

Jika `resource` tidak ada atau mengandung `:`, validator menghasilkan error:

```
Field '<name>' defaultValue.resource must be provided when source is 'idgen'
Field '<name>' defaultValue.resource is invalid: must be a string and must not contain ':'
```

## Mode `number`

Nomor urut dengan prefix berbasis tanggal. Cocok untuk nomor invoice, nomor SO, nomor PO.

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

### Atribut Mode `number`

| Atribut | Tipe | Wajib | Keterangan |
|---------|------|:-----:|-----------|
| `format` | string | ✓ | Format prefix tanggal. Valid: `text`, `yyyy`, `yyyymm`, `yyyymmdd` |
| `numDigits` | integer | ✓ | Jumlah digit nomor urut. Range valid: 1–12 |
| `prefix` | string | ✗ | Teks tetap di depan nomor, misalnya kode jenis dokumen. Lihat [Prefix Teks](#prefix-teks-prefix) |
| `separator` | string | ✗ | Pemisah antara prefix dan nomor. Tidak boleh mengandung `:` |
| `reserve` | boolean | ✗ | Reservasi nomor sebelum save (lihat penjelasan di bawah) |
| `ttl` | integer | ✗ | Time-to-live reservasi dalam detik. Wajib >= 10 jika `reserve: true` |

### Valid `format`

```
text, yyyy, yyyymm, yyyymmdd
```

Format `text` menghasilkan ID tanpa prefix tanggal (hanya nomor urut). Format `yyyy*` menambahkan prefix tanggal sesuai pola.

### Prefix Teks (`prefix`)

Atribut `prefix` menambahkan teks tetap di depan nomor, sehingga satu resource bisa menghasilkan kode berawalan seperti `VEND-001`. Dengan `format: "text"`, hasilnya berbentuk `{prefix}{separator}{nomor}`. Dengan format tanggal, prefix berada di depan prefix tanggal: `{prefix}{separator}{tanggal}{separator}{nomor}`.

```json
{
    "name": "vendor_code",
    "type": "text",
    "defaultValue": {
        "source": "idgen",
        "mode": "number",
        "resource": "vendor",
        "format": "text",
        "prefix": "VEND",
        "separator": "-",
        "numDigits": 3
    }
}
```

Contoh di atas menghasilkan `VEND-001`, `VEND-002`, dan seterusnya. Prefix kosong atau tidak diisi menghasilkan nomor tanpa teks di depan. Backend menolak prefix yang berisi karakter selain huruf, angka, `_`, dan `-`, atau yang lebih dari 32 karakter.

### Reservasi Nomor (`reserve`)

Jika `reserve: true`, backend mereservasi nomor urut sebelum user men-submit form. Reservasi punya TTL untuk auto-release jika user batal save.

Aturan reservasi:

| Aturan | Validasi |
|--------|----------|
| Maksimal 1 field per page yang boleh memakai `reserve: true` | Error jika >1: `"Only one field per page may use defaultValue.reserve=true. Found: <names>"` |
| `ttl` >= 10 jika `reserve: true` | Error jika < 10: `"Field '<name>' defaultValue.ttl must be an integer >= 10 when reserve is true"` |
| `reserve` hanya untuk `mode: "number"` | Error jika dipakai di mode lain: `"Field '<name>' defaultValue.reserve is only supported when mode is 'number'"` |

## Mode `pin`

PIN angka acak n-digit.

```json
{
    "name": "verification_pin",
    "type": "text",
    "defaultValue": {
        "source": "idgen",
        "mode": "pin",
        "resource": "verification",
        "digits": 6
    }
}
```

| Atribut | Tipe | Default | Keterangan |
|---------|------|---------|-----------|
| `digits` | integer | `6` | Jumlah digit PIN. Range valid: 4–12 |

Validator menolak nilai di luar range:

```
Field '<name>' defaultValue.digits must be an integer between 4 and 12
```

## Mode `code`, `serial`, `random`

Tiga mode ini memakai atribut `pattern` untuk menentukan format output.

```json
{
    "name": "item_code",
    "type": "text",
    "defaultValue": {
        "source": "idgen",
        "mode": "code",
        "resource": "item",
        "pattern": "ITM-####"
    }
}
```

| Atribut | Tipe | Wajib | Keterangan |
|---------|------|:-----:|-----------|
| `pattern` | string | ✓ | Pattern format. Non-empty |

Validator menolak pattern kosong:

```
Field '<name>' defaultValue.pattern must be a non-empty string when mode is '<mode>'
```

Sintaks pattern (mis. `#` untuk digit acak, `?` untuk huruf) bergantung pada implementasi backend generator. Lihat dokumentasi backend (RDF) untuk sintaks lengkap.

## IdGen pada Field Detail `one`

Field di `details[].fields[]` boleh memakai `defaultValue` IdGen, dan validator memeriksanya dengan aturan yang sama seperti field page. Pesan error-nya diawali lokasi detail, misalnya `[business-partner].details[vendor] field 'vendor_code' ...`.

Pada [detail `one`](./master-detail.md#detail-satu-baris-one), nilai IdGen diminta saat detail ditambahkan lewat tombol tambah, bukan saat form dibuka. Pada contoh vendor di atas, kode `VEND-001` terisi ketika peran vendor ditambahkan. Bila field memakai `reserve: true`, alurnya mengikuti mode add. Nomor direservasi saat detail ditambahkan, reservasi dibatalkan bila detail dilepas sebelum simpan atau form ditutup tanpa simpan, dan reservasi dikonfirmasi saat simpan berhasil.

Pada mode duplicate, field IdGen detail `one` meminta nilai baru dan tidak menyalin nilai record asal.

## Tabel Ringkasan Validasi IdGen

| Aturan | Trigger Error |
|--------|---------------|
| `source` harus `"idgen"` | Tidak dipicu (validator hanya check jika `source == "idgen"`) |
| Field type harus `number`/`text`/`textarea` | Error untuk tipe lain |
| `mode` wajib + valid | Error jika tidak ada atau invalid |
| `resource` wajib + tidak mengandung `:` | Error jika absent atau invalid |
| `mode: number` butuh `format` valid | Error jika tidak ada atau invalid |
| `mode: number` butuh `numDigits` integer 1–12 | Error jika tidak ada atau out of range |
| `mode: number` `separator` tidak boleh mengandung `:` | Error jika mengandung |
| `reserve: true` hanya untuk `mode: number` | Error jika dipakai di mode lain |
| `reserve: true` butuh `ttl` >= 10 | Error jika < 10 |
| `mode: pin` butuh `digits` integer 4–12 | Error jika out of range |
| `mode: code/serial/random` butuh `pattern` non-empty | Error jika absent atau kosong |
| Maks 1 field dengan `reserve: true` per page | Error jika >1 |

## Contoh Penggunaan Praktis

### Nomor Invoice dengan Reservasi

```json
{
    "name": "invoice_no",
    "type": "text",
    "label": "Nomor Invoice",
    "readonly": true,
    "defaultValue": {
        "source": "idgen",
        "mode": "number",
        "resource": "invoice",
        "format": "yyyymm",
        "numDigits": 4,
        "separator": "/",
        "reserve": true,
        "ttl": 120
    }
}
```

Menghasilkan nomor: `202503/0001`, `202503/0002`, dst. Reservasi auto-release setelah 120 detik jika form tidak di-submit.

### PIN Verifikasi 6 Digit

```json
{
    "name": "otp_code",
    "type": "text",
    "label": "OTP",
    "defaultValue": {
        "source": "idgen",
        "mode": "pin",
        "resource": "otp",
        "digits": 6
    }
}
```

### Kode Item dengan Pattern

```json
{
    "name": "item_code",
    "type": "text",
    "label": "Kode Item",
    "defaultValue": {
        "source": "idgen",
        "mode": "code",
        "resource": "item",
        "pattern": "ITM-####"
    }
}
```

---

← [`data-source.md`](./data-source.md) | [Selanjutnya: `field-rows.md`](./field-rows.md) →
