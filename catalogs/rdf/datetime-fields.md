# Field DateTime (`dateTimeFields`)

Object yang mendeklarasikan field bertipe waktu (`date`, `timestamp`, `timestamptz`, `time`) agar
diproses runtime saat operasi tulis dan baca. Blok ini menangani **pemrosesan nilai waktu**:
normalisasi input sebelum masuk database dan pemformatan nilai saat dikembalikan ke klien. Berbeda
dari [`fieldValidation`](./field-validation.md) yang mengurus aturan validasi (`required`,
`maxLength`, dan sejenisnya), `dateTimeFields` mengurus konversi bentuk nilainya.

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
| `type` | enum | Ya | `date`, `timestamp`, `timestamptz`, atau `time`. `datetime` diterima sebagai alias legacy dari `timestamp` (termasuk kolom `DATETIME` MySQL, dinormalisasi ke `timestamp` saat introspeksi) |
| `format` | string | Tidak | Hanya untuk `type: "time"`, berisi `HH:mm`, `HH:mm:ss`, atau `HH:mm:ss.SSS`. Tidak berlaku untuk `date`, `timestamp`, dan `timestamptz` (lihat [Satu Standar per Aplikasi](#satu-standar-per-aplikasi)) |

Field yang tidak terdaftar di `dateTimeFields` tidak diproses sebagai waktu: nilainya diteruskan
apa adanya ke database tanpa normalisasi.

## Konfigurasi Zona dan Format

Perilaku field `timestamp`, `timestamptz`, dan `date` diatur oleh tiga parameter environment di
`config/db-connection.env`:

| Parameter | Default | Mengatur |
|-----------|---------|----------|
| `TIMEZONE` | `UTC` | Zona IANA (mis. `Asia/Jakarta`) yang dipakai menafsirkan input tanpa zona untuk `timestamp`/`timestamptz` dan mengisi kolom audit (`created_at`, `updated_at`). Tidak memengaruhi `date` |
| `DATEFORMAT` | `yyyy-MM-dd` | Pola input/output semua field `date` |
| `DATETIMEFORMAT` | `yyyy-MM-dd HH:mm:ss.SSS` | Pola input/output semua field `timestamp`, ditulis sebagai `<pola tanggal> <pola waktu>`. Untuk `timestamptz`, pola ini hanya berlaku sebagai bentuk input lokal; output `timestamptz` selalu ISO UTC |

Pola tanggal yang didukung: `yyyy-MM-dd`, `dd/MM/yyyy`, `dd-MM-yyyy`, `MM/dd/yyyy`, `yyyy/MM/dd`.
Pola waktu yang didukung: `HH:mm`, `HH:mm:ss`, `HH:mm:ss.SSS`. `DATETIMEFORMAT` harus berbentuk
salah satu pola tanggal diikuti spasi lalu salah satu pola waktu, misalnya `dd/MM/yyyy HH:mm:ss`.

`TIMEZONE` yang bukan nama zona IANA yang dikenal, atau `DATEFORMAT`/`DATETIMEFORMAT` dengan pola
di luar daftar di atas, membuat server gagal start dengan pesan yang menyebut nilai mana yang
tidak valid dan pola apa saja yang didukung.

## Satu Standar per Aplikasi

Ketiga parameter di atas adalah satu-satunya standar tanggal-waktu aplikasi. Satu field bertipe
sama selalu keluar dalam bentuk yang sama di semua endpoint dan semua dialect database, sehingga
klien tidak perlu tahu endpoint mana yang memformat dan mana yang tidak.

Bentuk yang sama berlaku pada respons `create`, `update`, `first`, `read`, `datatables`,
`lookup`, `aggregate`, `delete`, `restore`, serta `create-composite`, `update-composite`, dan
`read-composite` (header maupun detail). Aturannya juga sama di PostgreSQL, MySQL, SQLite, dan
Oracle.

Tidak ada pola per field. Payload yang menulis `format` pada entry `dateTimeFields` bertipe
`date`, `timestamp`, atau `timestamptz`, atau `constraints.format` pada entry `fieldValidation`
bertipe tanggal, ditolak saat `payload validate` dan `endpoint create`:

```
dateTimeFields['ship_date'].format in sales-order.json is not supported for type 'date'; the pattern always comes from DATEFORMAT. Remove "format"
```

Aturan ini berlaku di blok utama maupun di `masterDetail.detailConfig`. Hapus `format` tersebut,
lalu atur pola lewat `DATEFORMAT` atau `DATETIMEFORMAT`. Field `time` tetap boleh memakai `format`
karena tipe ini tidak diatur pola global.

### Header Pola Aktif

Setiap respons API membawa pola yang sedang berlaku, sehingga klien di luar frontend hasil
generate dapat membaca nilai tanpa konfigurasi manual:

| Header | Isi | Contoh |
|--------|-----|--------|
| `X-Timezone` | Nilai `TIMEZONE` | `Asia/Jakarta` |
| `X-Date-Format` | Nilai `DATEFORMAT` | `dd/MM/yyyy` |
| `X-DateTime-Format` | Nilai `DATETIMEFORMAT` | `dd/MM/yyyy HH:mm:ss.SSS` |

Ketiga header tercantum di `Access-Control-Expose-Headers`, sehingga JavaScript di browser dapat
membacanya pada request lintas origin.

## Bentuk Input yang Diterima

Untuk `type: "timestamp"` dan `type: "timestamptz"`, tiga bentuk berikut diterima:

| Bentuk | Contoh | Interpretasi |
|--------|--------|--------------|
| ISO berzona (`Z` atau offset) | `2026-09-18T07:00:00+07:00` | Momen eksplisit; offset menentukan momennya dan tidak pernah ditimpa `TIMEZONE` |
| ISO/kanonik tanpa zona (pemisah `T` atau spasi) | `2026-09-18T07:00:00`, `2026-09-18 07:00:00.483` | Wall-clock zona aplikasi (`TIMEZONE`) |
| Pola `DATETIMEFORMAT` | `18/09/2026 07:00:00` (dengan `DATETIMEFORMAT=dd/MM/yyyy HH:mm:ss`) | Wall-clock zona aplikasi |

Bentuk ISO selalu diterima di samping pola `DATETIMEFORMAT`, karena ISO tidak ambigu. Bentuk lain
ditolak `400` dengan `code: INVALID_DATETIME`, termasuk:

| Input | Alasan penolakan |
|-------|------------------|
| `2026-09-18` atau `18/09/2026` pada field `timestamp`/`timestamptz` | Tanggal saja tidak memuat jam. Backend tidak melengkapinya dengan `00:00:00` |
| `18/09/2026 07:00` dengan `DATETIMEFORMAT=dd/MM/yyyy HH:mm:ss` | Tidak cocok dengan pola aktif maupun ISO |
| `31/02/2026 07:00:00` | Tanggal kalender tidak nyata |
| `18/09/2026 25:00:00` | Jam di luar rentang |

Penolakan tanggal saja mencegah nilai yang salah maksud tercatat sebagai momen sah. Klien yang
mengirim nilai tanggal ke field tanggal-waktu mendapat error, bukan jam `00:00` yang tidak pernah
dimaksudkan.

Pola `dd/MM/yyyy` dan `MM/dd/yyyy` tidak dapat dibedakan untuk tanggal 1 sampai 12: input
`05/07/2026` sah pada kedua pola. Karena itu klien wajib memakai pola aktif dari header respons
atau `appConfig`, atau mengirim ISO.

### Tipe `date`: Tanggal Kalender

Field `date` tidak mengenal komponen waktu dan tidak terpengaruh `TIMEZONE`. Selalu menerima
`YYYY-MM-DD`, ditambah pola `DATEFORMAT`:

| Input | Hasil |
|-------|-------|
| `2026-07-03` | `2026-07-03` |
| `03/07/2026` (dengan `DATEFORMAT=dd/MM/yyyy`) | `2026-07-03` |
| `07/03/2026` (dengan `DATEFORMAT=dd/MM/yyyy`) | `2026-03-07`, karena pola aktif dibaca apa adanya |
| `31/02/2026` | `400` (tanggal kalender tidak nyata) |
| `2026-07-03T14:30:00` | `400` (membawa komponen waktu) |
| `2026-07-03T00:00:00.000Z` | `400` (membawa komponen waktu, mis. hasil `toISOString()`) |
| `2026-07-03 14:30:00` | `400` (membawa komponen waktu) |

Alasan penolakan tegas ini: nilai seperti `2026-07-03T14:30:00` dan `2026-07-03 14:30:00` bermakna
sama, sehingga keduanya harus diperlakukan sama. Menolak (bukan memotong) juga melindungi dari bug
pergeseran tanggal (off-by-one) yang lazim terjadi saat frontend mengirim tanggal via
`Date.toISOString()`. Panduan frontend: kirim string `YYYY-MM-DD` langsung, mis. dari
`<input type="date">`, dan hindari `Date.toISOString()` untuk tanggal.

## Normalisasi Zona Waktu

`timestamp` dan `timestamptz` menafsirkan input dengan cara berbeda, meski bentuk input yang
diterima sama:

| Tahap | `timestamp` (tanpa zona) | `timestamptz` (berzona) |
|-------|--------------------------|--------------------------|
| Input berzona | Dikonversi ke `TIMEZONE` aplikasi dulu, lalu komponen tanggal/jam hasil konversi itu yang ditulis | Offset pada input dipakai apa adanya untuk menentukan momen; tidak pernah ditimpa `TIMEZONE` |
| Input tanpa zona | Komponen ditafsirkan sudah berada di `TIMEZONE` aplikasi dan disimpan apa adanya | Ditafsirkan sebagai wall-clock `TIMEZONE` aplikasi, lalu dikonversi ke momen |
| Nilai tersimpan | Komponen wall-clock, tanpa informasi zona | Momen absolut (PostgreSQL menormalisasinya secara internal, nama zona asal tidak disimpan) |
| Bentuk output | Pola `DATETIMEFORMAT` | Selalu ISO UTC, tidak terpengaruh `TIMEZONE` maupun `DATETIMEFORMAT` |

Hasil tidak berubah ketika zona mesin tempat server berjalan berbeda dari `TIMEZONE`, maupun
ketika zona sesi database berbeda.

### Kasus Satu Zona

Dengan `TIMEZONE=Asia/Jakarta` dan `DATETIMEFORMAT=dd/MM/yyyy HH:mm:ss`, body
`{"clockin":"12/09/2026 05:00:00"}` berarti pukul 05.00 waktu Jakarta:

| Tahap | `timestamp` | `timestamptz` |
|-------|-------------|----------------|
| Nilai tersimpan | `2026-09-12 05:00:00` (tanpa zona) | Momen `2026-09-12 05:00:00+07:00`, setara `2026-09-11 22:00:00Z` |
| Response | `12/09/2026 05:00:00` | `2026-09-11T22:00:00.000Z` |
| Tampilan frontend di komputer zona Jakarta | `12/09/2026 05:00:00` | `12/09/2026 05:00:00` |

Pada `timestamp`, response sudah berpola lokal. Pada `timestamptz`, response selalu ISO UTC, lalu
frontend mengonversinya ke zona komputer pembaca sebelum menerapkan `DATETIMEFORMAT`. Hasil
tampilannya sama di komputer zona Jakarta, tetapi semantik penyimpanannya berbeda: `timestamp`
menyimpan komponen jam, `timestamptz` menyimpan momen.

### Kasus Absensi Surabaya dan Balikpapan

Server memakai `TIMEZONE=Asia/Jakarta`, kolom `clockin` bertipe `timestamptz`. Karyawan Surabaya
(`Asia/Jakarta`, UTC+7) dan Balikpapan (`Asia/Makassar`, UTC+8) sama-sama clock-in pukul 07.00
waktu setempat, masing-masing mengirim ISO dengan offset lokalnya:

```json
{"employee_id":"EMP-001","clockin":"2026-09-18T07:00:00+07:00"}
{"employee_id":"EMP-002","clockin":"2026-09-18T07:00:00+08:00"}
```

Offset eksplisit menentukan momennya, sehingga keduanya tersimpan sebagai momen berbeda meski jam
lokalnya sama pukul 07.00:

| Karyawan | Momen tersimpan (UTC) | Response `read` (ISO UTC) |
|----------|------------------------|-----------------------------|
| EMP-001 (Surabaya) | `2026-09-18 00:00:00Z` | `2026-09-18T00:00:00.000Z` |
| EMP-002 (Balikpapan) | `2026-09-17 23:00:00Z` | `2026-09-17T23:00:00.000Z` |

Karyawan Balikpapan clock-in satu jam lebih dahulu secara absolut. Bila klien mengirim tanpa zona,
misalnya `{"clockin":"18/09/2026 07:00:00"}`, backend menafsirkannya sebagai `TIMEZONE` aplikasi
(Jakarta) sehingga hasilnya `2026-09-18T00:00:00.000Z`, setara `+07:00`. Klien yang menangani
karyawan lintas zona harus mengirim ISO ber-offset atau ISO UTC, karena backend tidak dapat menebak
zona dari string tanpa penanda.

Query baca tidak memerlukan join ke tabel zona atau kolom timezone tambahan; kolom `timestamptz`
sudah cukup menyimpan kedua momen dengan benar. Response selalu ISO UTC agar frontend dapat
mengonversinya sendiri ke zona pembaca:

| Data | Dilihat di komputer Surabaya (UTC+7) | Dilihat di komputer Balikpapan (UTC+8) |
|------|----------------------------------------|-------------------------------------------|
| Clock-in EMP-001 | `18/09/2026 07:00:00` | `18/09/2026 08:00:00` |
| Clock-in EMP-002 | `18/09/2026 06:00:00` | `18/09/2026 07:00:00` |

Input pukul 07.00 tampil kembali pukul 07.00 pada komputer asal karena zona pembaca sama dengan
zona input. Bila dibuka di zona lain, jam yang tampil berbeda meski momennya tetap sama.

### Milidetik

Presisi milidetik dipertahankan dari input sampai response. Input `2026-09-12T05:00:00.483+07:00`
tersimpan dan dikembalikan dengan `.483`. Fraksi yang lebih halus dari milidetik, misalnya
mikrodetik dari default database `now()`, dipotong ke 3 digit pada response.

Milidetik hanya hilang dari response bila `DATETIMEFORMAT` tidak memuat `.SSS`, misalnya
`dd/MM/yyyy HH:mm:ss`. Itu keputusan tampilan yang disengaja. Field yang dipakai
`concurrency.compare: "timestamp"` (lihat [`concurrency.md`](./concurrency.md)) membutuhkan
`DATETIMEFORMAT` dengan `.SSS`. Token `versionColumn` dari response `first` dibandingkan pada
presisi milidetik; tanpa `.SSS`, token kehilangan presisi dan update ditolak sebagai konflik.
Server mencatat peringatan sekali untuk kondisi ini.

## Penolakan DST

Input tanpa zona yang jatuh pada celah DST (jam yang dilompati) atau tumpang-tindih DST (jam yang
terjadi dua kali) di `TIMEZONE` aplikasi ditolak dengan `400`, `code: INVALID_DATETIME`. Berlaku
untuk `timestamp` dan `timestamptz`, baik pada operasi tulis maupun filter `where`. Contoh pada
`TIMEZONE=America/New_York`:

| Input | Kondisi | Pesan |
|-------|---------|-------|
| `08/03/2026 02:30:00` | Celah DST, jam ini tidak pernah ada | `Datetime value '08/03/2026 02:30:00' does not exist in time zone 'America/New_York' due to a DST transition` |
| `01/11/2026 01:30:00` | Tumpang-tindih DST, jam ini terjadi dua kali | `Datetime value '01/11/2026 01:30:00' is ambiguous in time zone 'America/New_York' (possible offsets -04:00 and -05:00); send an ISO value with an explicit offset` |

Backend tidak menebak salah satu offset. Klien yang memerlukan momen pasti pada tanggal-tanggal
ini mengirim ISO ber-offset, yang selalu diterima apa adanya tanpa terpengaruh DST.

## Bentuk Kanonik

Nilai ditulis ke database dalam bentuk berikut, identik lintas dialect. Bentuk ini juga menjadi
bentuk response bila ketiga parameter memakai nilai default.

| Tipe | Bentuk kanonik | Contoh |
|------|----------------|--------|
| `timestamp` | `YYYY-MM-DD HH:mm:ss.SSS` | `2026-07-03 10:55:53.483` |
| `timestamptz` | `YYYY-MM-DDTHH:mm:ss.SSSZ` (ISO UTC) | `2026-07-03T10:55:53.483Z` |
| `date` | `YYYY-MM-DD` | `2026-07-03` |
| `time` | `HH:mm:ss` | `10:55:00` |

## Dukungan per Dialect

| Tipe SDF | PostgreSQL | MySQL | SQLite | Oracle |
|----------|------------|-------|--------|--------|
| `date` | `DATE` | `DATE` | `DATE` | `DATE` |
| `timestamp` | `TIMESTAMP` | `DATETIME(3)` | `TIMESTAMP` | `TIMESTAMP` |
| `timestamptz` | `TIMESTAMPTZ` | ditolak | ditolak | ditolak |

MySQL memakai `DATETIME(3)` agar wall-clock tersimpan tanpa konversi zona sesi, milidetik
bertahan, dan tidak berbatas tahun 2038. Dialect tanpa tipe berzona menolak `timestamptz` saat
DDL dibuat, tanpa fallback diam-diam ke tipe tanpa zona (lihat
[`sdf/field-types.md`](../sdf/field-types.md#tipe-timestamptz)).

Kolom audit `created_at` dan `updated_at` diisi wall-clock `TIMEZONE` oleh runtime di semua
dialect, sehingga nilainya tidak bergantung pada zona sesi database. Kolom ini tidak perlu
didaftarkan di `dateTimeFields`: bila ikut muncul di response, nilainya tetap mengikuti
`DATETIMEFORMAT` seperti field `timestamp` lain. Sebaliknya, `DEFAULT now()` pada DDL MySQL dan
SQLite dijalankan database dan mengikuti zona sesi database.

Pada Oracle, server yang berjalan di zona mesin ber-DST dan berbeda dari `TIMEZONE` tidak dapat
membaca ulang jam yang jatuh pada celah DST zona mesin tersebut. Server mencatat peringatan saat
start untuk kondisi ini. Jalankan server Oracle dengan zona mesin `UTC` atau sama dengan
`TIMEZONE`.

## Filter `where` pada `read`, `first`, dan `datatables`

Nilai filter untuk field yang terdaftar di `dateTimeFields` diterima dalam bentuk yang sama
dengan jalur tulis, mencakup operator perbandingan biasa, `BETWEEN`, dan `IN` (termasuk pada
`where` bersarang). Contoh: filter `clockin = "18/09/2026 07:00:00"` pada kolom `timestamptz`
dengan `TIMEZONE=Asia/Jakarta` dikonversi ke momen `2026-09-18T00:00:00.000Z` sebelum masuk SQL,
persis seperti nilai yang sama diproses pada jalur `create`/`update`.

Aturan penolakan juga sama. Tanggal saja pada filter field `timestamp` ditolak, sehingga rentang
satu hari ditulis dengan jam eksplisit, misalnya `BETWEEN ["18/09/2026 00:00:00", "18/09/2026
23:59:59"]`. Input filter yang tidak valid ditolak `400` dengan `code: INVALID_DATETIME`.

Filter dan urutan `sort_columns` tetap memakai nilai database. Pemformatan terjadi setelah query,
sehingga pola aktif tidak memengaruhi baris yang cocok maupun urutannya. Field di luar
`dateTimeFields`, operator `IS NULL`/`IS NOT NULL`, dan `LIKE`/`NOT LIKE` (pencarian teks) tidak
dinormalisasi.

## Interaksi dengan `autoGenerate`

Bila field waktu di [`fieldValidation`](./field-validation.md) memiliki constraint
`autoGenerate: true` dan request tidak menyertakan nilainya, runtime mengisi otomatis dengan waktu
saat ini. Nilai auto-generate ini juga melewati normalisasi `dateTimeFields`, sehingga tersimpan
dalam bentuk kanonik. Field `date` diisi tanggal hari ini menurut `TIMEZONE`, bukan tanggal UTC.
Kombinasi lazim untuk kolom `timestamp NOT NULL DEFAULT now()`:

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

`dateTimeFields` juga dipakai saat membentuk response, dengan aturan yang berbeda per tipe:

- `timestamp`: selalu diformat dengan `DATETIMEFORMAT`, termasuk bila pola itu masih default.
  Fraksi detik dipotong ke 3 digit.
- `timestamptz`: selalu ISO UTC, terlepas dari `TIMEZONE` maupun `DATETIMEFORMAT`.
- `date`: selalu diformat dengan `DATEFORMAT` dan tidak pernah bergeser oleh `TIMEZONE`.
- `time`: memakai `format` pada entry bila ada, atau `HH:mm:ss`.

Nilai `null` selalu dikembalikan sebagai `null`. Aturan ini berlaku untuk detail master-detail
dengan cara yang sama, memakai tipe kolom detail yang tercatat di `masterDetail.detailConfig`.

Contoh satu record dengan `DATEFORMAT=dd/MM/yyyy` dan `DATETIMEFORMAT=dd/MM/yyyy HH:mm:ss.SSS`:

```json
{
    "order_date":   "20/08/2026",
    "posted_at":    "12/09/2026 05:06:07.483",
    "confirmed_at": "2026-09-11T22:06:07.483Z"
}
```

Nilai yang sama keluar dari `first`, `read`, `datatables`, `lookup`, respons `create`/`update`,
dan `read-composite`.

### Keselarasan dengan Frontend Hasil Generate

Frontend hasil generate membaca nilai tanggal dengan pola yang sama dengan backend. `payload
migrate` menyalin `DATEFORMAT` dan `DATETIMEFORMAT` ke `appConfig.dateFormat` dan
`appConfig.dateTimeFormat`, dan menandai field `timestamptz` dengan `temporalType`. Frontend tidak
dapat mendeteksi pola yang tidak selaras, sehingga setiap perubahan `DATEFORMAT` atau
`DATETIMEFORMAT` wajib diikuti `payload migrate` dan generate frontend ulang. Lihat
[`catalogs/udf/app-config.md`](../udf/app-config.md#pola-tanggal-dateformat-dan-datetimeformat).

## Format Error

Input waktu yang tidak valid, pada body maupun filter `where`, menghasilkan HTTP `400` di semua
endpoint:

```json
{
    "success": false,
    "error": "Invalid datetime",
    "code": "INVALID_DATETIME",
    "message": "Invalid datetime value '18/09/2026': type 'timestamp' requires a time component; a date-only value is not accepted. Expected an ISO 8601 value or format 'dd/MM/yyyy HH:mm:ss.SSS'",
    "timestamp": "2026-07-03T10:30:00.000Z"
}
```

Pesan menyebut nilai yang diterima dan pola yang diharapkan. Untuk field `date`, pesan menyebut
`expected 'YYYY-MM-DD' or format '<pola>'`. Untuk penolakan DST, lihat pesan spesifik di bagian
[Penolakan DST](#penolakan-dst) di atas.

## Dihasilkan oleh `payload generate`

Saat payload diturunkan dari introspeksi database, kolom bertipe waktu otomatis masuk
`dateTimeFields`:

- `DATE` → `{ "type": "date" }`
- `TIMESTAMP WITHOUT TIME ZONE` / `TIMESTAMP` → `{ "type": "timestamp" }`
- `TIMESTAMP WITH TIME ZONE` → `{ "type": "timestamptz" }`
- `TIME` → `{ "type": "time" }`
- `DATETIME` (MySQL) → `{ "type": "timestamp" }`

Entry ditulis **tanpa `format`**, sehingga field waktu hasil generate langsung mengikuti pola
global tanpa konfigurasi tambahan.

## Hubungan dengan `fieldValidation`

Kedua blok bekerja bersama namun terpisah tugasnya:

| Blok | Tugas |
|------|-------|
| `dateTimeFields` | Pemrosesan nilai waktu: normalisasi input, pemformatan output |
| `fieldValidation` | Validasi constraint: `required`, `autoGenerate`, `min`, `max`, `before`, `after`, dan sejenisnya |

Untuk satu field waktu, umumnya keduanya hadir: `dateTimeFields` mengurus bentuk waktunya,
`fieldValidation` mengurus aturan wajib-isi dan auto-fill.

---

**Lihat juga**: [`field-validation.md`](./field-validation.md) · [`concurrency.md`](./concurrency.md) · [`data-source.md`](./data-source.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
