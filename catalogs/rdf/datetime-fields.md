# Field DateTime (`dateTimeFields`)

Object yang mendeklarasikan field bertipe waktu (date, timestamp, time) agar diproses runtime
saat operasi tulis dan baca. Blok ini menangani **pemrosesan nilai waktu**: normalisasi input
sebelum masuk database dan pemformatan nilai saat dikembalikan ke klien. Berbeda dari
[`fieldValidation`](./field-validation.md) yang mengurus aturan validasi (`required`, `maxLength`,
dan sejenisnya), `dateTimeFields` mengurus konversi bentuk nilainya.

```json
{
    "dateTimeFields": {
        "visit_date": {
            "type": "timestamp"
        }
    }
}
```

## Struktur Entry

Key object adalah nama field (harus ada di `fieldName`). Value berisi:

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `type` | enum | Ya | `date`, `timestamp`, atau `time`. Kolom `datetime` (MySQL) dinormalisasi ke `timestamp` |
| `format` | string | Tidak | Pola input yang ditegakkan. Ada `format` → mode strict; tanpa `format` → mode lenient (lihat di bawah) |

Field yang tidak terdaftar di `dateTimeFields` tidak diproses sebagai waktu: nilainya diteruskan
apa adanya ke database tanpa normalisasi.

## Dua Mode: Lenient dan Strict

Perilaku ditentukan oleh ada atau tidaknya `format`.

| Mode | Pemicu | Perilaku |
|------|--------|----------|
| Lenient | `format` tidak ada | Menerima beragam bentuk input yang lazim (ISO 8601, separator spasi, date-only), lalu menormalisasi ke bentuk kanonik. Bentuk yang tidak dikenali ditolak dengan `400` |
| Strict | `format` ada | Input wajib cocok persis dengan pola `format`. Tidak cocok ditolak dengan `400` |

Sejak `payload generate` diperbarui, generator **tidak lagi menulis `format` secara otomatis**
untuk kolom waktu, sehingga default field datetime hasil generate adalah **lenient**. `format`
hanya ditulis bila memang dikehendaki penguncian pola input tertentu (mis. dari layer UDF).

## Mode Lenient

Berlaku saat `format` tidak ada. Untuk `type: "timestamp"`, bentuk input yang diterima dan hasil
normalisasinya:

| Input | Hasil tersimpan |
|-------|-----------------|
| `2026-07-03T10:55:53.262Z` | `2026-07-03 10:55:53` |
| `2026-07-03T17:55:53+07:00` | `2026-07-03 10:55:53` |
| `2026-07-03T10:55:53` | `2026-07-03 10:55:53` |
| `2026-07-03T10:55` | `2026-07-03 10:55:00` |
| `2026-07-03 10:55` | `2026-07-03 10:55:00` |
| `2026-07-03 10:55:53.262` | `2026-07-03 10:55:53` |
| `2026-07-03` | `2026-07-03 00:00:00` |

Bentuk yang **ditolak** dengan `400`: epoch milidetik (angka atau string angka seperti
`1751537753262`), pola tanggal ambigu seperti `dd/MM/yyyy` tanpa penanda, dan string yang tidak
dapat di-parse.

### Normalisasi Zona Waktu

Aturan konversi untuk `type: "timestamp"`:

- Input yang **membawa zona** (`Z` atau offset seperti `+07:00`) dikonversi ke UTC, lalu ditulis
  sebagai `YYYY-MM-DD HH:mm:ss`. Contoh: `2026-07-03T17:55:53+07:00` dan `2026-07-03T10:55:53Z`
  keduanya tersimpan `2026-07-03 10:55:53`, karena memang menunjuk momen yang sama.
- Input **tanpa zona** (separator spasi, datetime-local, date-only) disimpan apa adanya sebagai
  wall-clock, tanpa konversi. Ini menjaga kompatibilitas dengan data yang mengirim jam lokal
  tanpa penanda zona.
- Milidetik dipangkas ke detik.

Kolom `timestamp` demikian menyimpan momen dalam UTC untuk input yang menyebutkan zonanya. Untuk
kebutuhan garis waktu internasional yang dijaga penuh di level database, gunakan tipe kolom
`timestamptz` secara eksplisit (di luar cakupan default `dateTimeFields`).

### Tipe `date`: hanya `YYYY-MM-DD`

Berbeda dari `timestamp`, mode lenient untuk `type: "date"` bersifat ketat: hanya menerima tanggal
murni `YYYY-MM-DD`. Semua input yang membawa komponen waktu ditolak dengan `400`, tidak dipotong
diam-diam.

| Input | Hasil |
|-------|-------|
| `2026-07-03` | `2026-07-03` |
| `2026-07-03T14:30:00` | `400` (membawa waktu) |
| `2026-07-03T00:00:00.000Z` | `400` (membawa waktu, mis. hasil `toISOString()`) |
| `2026-07-03 14:30:00` | `400` (membawa waktu) |
| `03/07/2026` | `400` (bukan `YYYY-MM-DD`, kecuali dideklarasikan strict) |

Alasan penolakan tegas ini: nilai seperti `2026-07-03T14:30:00` dan `2026-07-03 14:30:00` bermakna
sama, sehingga keduanya harus diperlakukan sama. Menolak (bukan memotong) juga melindungi dari bug
pergeseran tanggal (off-by-one) yang lazim terjadi saat frontend mengirim tanggal via
`Date.toISOString()`.

Panduan frontend untuk field `date`: kirim string `YYYY-MM-DD` langsung, mis. dari
`<input type="date">` (value-nya sudah `YYYY-MM-DD`) atau bentuk manual dari komponen tanggal lokal.
Hindari `Date.toISOString()` untuk tanggal. Bila memang perlu menerima format lain seperti
`dd/MM/yyyy`, deklarasikan `format` secara eksplisit (mode strict, lihat bagian berikut).

### Bentuk Kanonik

Output normalisasi selalu memakai bentuk berikut, identik lintas dialect database:

| Tipe | Bentuk kanonik | Contoh |
|------|----------------|--------|
| `timestamp` | `YYYY-MM-DD HH:mm:ss` | `2026-07-03 10:55:53` |
| `date` | `YYYY-MM-DD` | `2026-07-03` |
| `time` | `HH:mm:ss` | `10:55:00` |

## Mode Strict

Berlaku saat `format` ada. Input wajib cocok persis dengan pola, termasuk bagian tanggalnya.
Bila tidak cocok, request ditolak dengan `400` dan pesan jelas.

Pola `format` yang didukung:

| Tipe | Pola yang dikenali |
|------|--------------------|
| `timestamp` | `yyyy-MM-dd HH:mm`, `yyyy-MM-dd HH:mm:ss`, `dd/MM/yyyy HH:mm`, `dd-MM-yyyy HH:mm`, `MM/dd/yyyy HH:mm`, `yyyy/MM/dd HH:mm` |
| `date` | `yyyy-MM-dd`, `dd/MM/yyyy`, `dd-MM-yyyy`, `MM/dd/yyyy`, `yyyy/MM/dd` |
| `time` | `HH:mm`, `HH:mm:ss` |

Contoh: dengan `format: "yyyy-MM-dd HH:mm"`, input `2026-07-03 10:55` diterima dan disimpan
`2026-07-03 10:55:00`, sedangkan input ISO `2026-07-03T10:55:53.262Z` ditolak dengan `400` karena
tidak cocok dengan pola yang ditegakkan.

## Interaksi dengan `autoGenerate`

Bila field waktu di [`fieldValidation`](./field-validation.md) memiliki constraint
`autoGenerate: true` dan request tidak menyertakan nilainya, runtime mengisi otomatis dengan waktu
saat ini. Nilai auto-generate ini juga melewati normalisasi `dateTimeFields`, sehingga tersimpan
dalam bentuk kanonik. Kombinasi lazim untuk kolom `timestamp NOT NULL DEFAULT now()`:

```json
{
    "dateTimeFields": {
        "visit_date": { "type": "timestamp" }
    },
    "fieldValidation": [
        {
            "name": "visit_date",
            "type": "datetime",
            "constraints": { "autoGenerate": true, "required": true }
        }
    ]
}
```

Dengan konfigurasi ini: field kosong terisi otomatis, field berisi dinormalisasi, dan input tak
valid ditolak `400`.

## Pemformatan Output

`dateTimeFields` juga dipakai saat membentuk response. Field waktu diformat mengikuti `format`
yang dideklarasikan; tanpa `format`, nilai kanonik dikembalikan apa adanya. Dengan begitu bentuk
nilai yang diterima klien konsisten dengan yang tersimpan.

## Format Error

Input waktu yang tidak valid pada operasi tulis menghasilkan HTTP `400`:

```json
{
    "success": false,
    "error": "Bad Request",
    "message": "Invalid datetime value '1751537753262' for expected format 'ISO 8601 / YYYY-MM-DD HH:mm'",
    "timestamp": "2026-07-03T10:30:00.000Z"
}
```

Pada mode strict, `message` menyebut pola `format` yang ditegakkan sehingga klien tahu bentuk yang
diharapkan.

## Dihasilkan oleh `payload generate`

Saat payload diturunkan dari introspeksi database, kolom bertipe waktu otomatis masuk
`dateTimeFields`:

- `DATE` → `{ "type": "date" }`
- `TIMESTAMP` / `TIMESTAMP WITH/WITHOUT TIME ZONE` → `{ "type": "timestamp" }`
- `TIME` → `{ "type": "time" }`
- `DATETIME` (MySQL) → `{ "type": "timestamp" }`

Entry ditulis **tanpa `format`** (mode lenient), sehingga field waktu hasil generate langsung
menerima input klien yang lazim tanpa konfigurasi tambahan.

## Hubungan dengan `fieldValidation`

Kedua blok bekerja bersama namun terpisah tugasnya:

| Blok | Tugas |
|------|-------|
| `dateTimeFields` | Pemrosesan nilai waktu: normalisasi input, pemformatan output, penentuan mode lenient/strict |
| `fieldValidation` | Validasi constraint: `required`, `autoGenerate`, `min`, `max`, `before`, `after`, dan sejenisnya |

Untuk satu field waktu, umumnya keduanya hadir: `dateTimeFields` mengurus bentuk waktunya,
`fieldValidation` mengurus aturan wajib-isi dan auto-fill.

---

**Lihat juga**: [`field-validation.md`](./field-validation.md) · [`data-source.md`](./data-source.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
