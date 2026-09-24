# Konfigurasi Aplikasi — `appConfig`

> Konfigurasi level aplikasi: nama, plugin target, base URL API, port server statis, format angka, dan pola tanggal.

## Sintaks

```json
{
    "appConfig": {
        "appName": "Contact Management",
        "appCode": "contact-mgmt",
        "plugin": "vanilla-js-basic",
        "apiBaseUrl": "http://localhost:3031/api/dbcontact",
        "port": 3000,
        "numberFormat": {
            "locale": "id-ID"
        },
        "dateFormat": "dd/MM/yyyy",
        "dateTimeFormat": "dd/MM/yyyy HH:mm:ss"
    }
}
```

## Properti

| Properti | Tipe | Default | Wajib | Keterangan |
|----------|------|---------|:-----:|-----------|
| `appName` | string | — | ✓ | Nama aplikasi yang ditampilkan di header/title UI |
| `appCode` | string | — | ✓ | Kode unik aplikasi dalam format kebab-case. Dipakai sebagai identifier internal |
| `plugin` | string | — | ✓ | ID plugin generator yang dipakai. Contoh: `vanilla-js-basic`, `vanilla-js-auth` |
| `apiBaseUrl` | string | — | ✓ | Base URL backend API. Digabung dengan `apiPath` setiap page menjadi endpoint final |
| `port` | integer | `3000` | ✗ | Port HTTP server untuk preview aplikasi (`app-start.bat`). Range valid: 1–65535 |
| `numberFormat` | object | — | ✗ | Konfigurasi karakter pemisah angka di seluruh aplikasi |
| `numberFormat.locale` | string | `"en-US"` | ✗ | Locale yang menentukan pemisah ribuan dan desimal. Nilai yang didukung: `"en-US"` (`12,500,000.00`) dan `"id-ID"` (`12.500.000,00`) |
| `numberFormat.currencyPrefix` | string | `"Rp "` | ✗ | Teks prefix yang ditampilkan di depan angka pada field ber-`format: "currency"` (dipakai `formatCurrency`) |
| `dateFormat` | string | `"yyyy-MM-dd"` | ✗ | Pola tanggal backend (`DATEFORMAT`). Nilai yang didukung: `yyyy-MM-dd`, `dd/MM/yyyy`, `dd-MM-yyyy`, `MM/dd/yyyy`, `yyyy/MM/dd` |
| `dateTimeFormat` | string | `"yyyy-MM-dd HH:mm:ss.SSS"` | ✗ | Pola tanggal-waktu backend (`DATETIMEFORMAT`), ditulis `<pola tanggal> <pola waktu>`. Pola waktu yang didukung: `HH:mm`, `HH:mm:ss`, `HH:mm:ss.SSS` |

`numberFormat` hanya mengatur karakter pemisah dan prefix mata uang. Jumlah digit desimal bukan pengaturan aplikasi, melainkan sifat tiap kolom: nilainya berasal dari skala kolom database (SDF `decimal:M,N`), ditulis `payload generate` sebagai `scale` di RDF, lalu diturunkan `payload migrate` menjadi `decimalPlaces` pada field UDF. RDF lama yang skalanya masih tersimpan di `precision` tetap terbaca sebagai fallback, disertai warning migrate agar `payload generate` dijalankan ulang. Lihat [`field-attributes.md`](./field-attributes.md#format-dan-decimalplaces-untuk-field-number).

## Pola Tanggal: `dateFormat` dan `dateTimeFormat`

Kedua properti ini wajib sama dengan `DATEFORMAT` dan `DATETIMEFORMAT` di `config/db-connection.env` backend. Frontend memakainya untuk membaca tanggal yang dikirim backend, lalu menampilkannya di form dan tabel list dengan pola yang sama. `payload migrate` menyalin nilainya dari file config, jadi pengguna tidak perlu menulisnya manual.

Setiap kali `DATEFORMAT` atau `DATETIMEFORMAT` diubah, `payload migrate` dan generate frontend wajib dijalankan ulang. Frontend tidak dapat mendeteksi pola yang tidak selaras dengan backend. Misalnya backend diubah ke `MM/dd/yyyy` sementara frontend masih `dd/MM/yyyy`: nilai `07/08/2026` akan terbaca sebagai tanggal lain tanpa pesan error.

Perilaku frontend terhadap nilai tanggal:

- Nilai dari backend hanya dibaca dalam bentuk ISO (`2026-07-30`) atau dalam pola yang berlaku. Nilai dalam bentuk lain tidak ditebak. Field dikosongkan, pesan peringatan muncul, dan form tidak dapat disimpan sampai tanggal dipilih lagi.
- Saat form Edit disimpan, field tanggal yang tidak diubah tidak ikut dikirim. Nilai di database tetap utuh, termasuk detik dan milidetik yang tidak tampil di form.
- Field `timestamp` yang diubah dikirim dengan detik, sedangkan milidetiknya menjadi `.000`.
- Field `timestamptz` tampil menurut zona waktu komputer pembaca, sehingga pengguna di Surabaya dan Balikpapan melihat jam yang berbeda untuk momen yang sama. Field `timestamp` dan `date` selalu tampil apa adanya, tanpa terpengaruh zona komputer.
- Nilai default `now` pada mode Add memakai jam komputer pembaca.

Pola tampilan satu field dapat diganti lewat properti `dateFormat` field, lihat [`field-attributes.md`](./field-attributes.md).

## Properti Tambahan dari Plugin

Plugin tertentu (mis. `vanilla-js-auth`) dapat menambahkan field di luar daftar inti. Field tambahan tidak divalidasi oleh validator umum tetapi oleh validator plugin yang bersangkutan. Contoh field plugin-specific:

| Field | Plugin | Keterangan |
|-------|--------|-----------|
| `auth.appCode` | `vanilla-js-auth` | Kode aplikasi yang dipakai saat call ke auth API |
| `auth.authApiUrl` | `vanilla-js-auth` | Endpoint server autentikasi |
| `auth.idleTimeoutMinutes` | `vanilla-js-auth` | Durasi idle sebelum auto-logout |

Lihat dokumentasi plugin masing-masing untuk field yang diakui.

## Aturan Validasi

| Aturan | Kondisi Error |
|--------|---------------|
| `appName` non-empty | `"appConfig.appName must be provided"` |
| `appCode` non-empty | `"appConfig.appCode must be provided"` |
| `plugin` non-empty | `"appConfig.plugin must be provided"` |
| `apiBaseUrl` non-empty | `"appConfig.apiBaseUrl must be provided"` |
| `port` harus integer 1–65535 | `"appConfig.port must be an integer between 1 and 65535, got <value>"` |
| `numberFormat` harus object jika ada | `"appConfig.numberFormat must be an object when provided, got <type>"` |
| `numberFormat.decimalPlaces` sudah dihapus | `"appConfig.numberFormat.decimalPlaces has been removed; the number of decimal places is now a field attribute. Set 'decimalPlaces' on the field instead (pages[].fields[].decimalPlaces, derived by \`payload migrate\` from the column scale)"` |
| `numberFormat.locale` harus salah satu locale yang didukung | `"appConfig.numberFormat.locale must be one of: en-US, id-ID, got '<value>'"` |
| `numberFormat.currencyPrefix` harus string jika ada | `"appConfig.numberFormat.currencyPrefix must be a string when provided, got <type>"` |
| `dateFormat` harus pola tanggal yang didukung | `"appConfig.dateFormat must be one of: yyyy-MM-dd, dd/MM/yyyy, dd-MM-yyyy, MM/dd/yyyy, yyyy/MM/dd, got '<value>'. It must match the backend DATEFORMAT in config/db-connection.env"` |
| `dateTimeFormat` harus `<pola tanggal> <pola waktu>` yang didukung | `"appConfig.dateTimeFormat must be '<date pattern> <time pattern>' with a date pattern from ... and a time pattern from ..., got '<value>'. It must match the backend DATETIMEFORMAT in config/db-connection.env"` |

Properti `port` dengan nilai string secara eksplisit ditolak oleh validator untuk mencegah shell injection lewat template `app-start.bat`.

## Hubungan dengan `apiPath` Page

Endpoint final yang dipanggil aplikasi terbentuk dari penggabungan `apiBaseUrl` (level app) + `apiPath` (level page):

```
http://localhost:3031/api/dbcontact + /contact = http://localhost:3031/api/dbcontact/contact
```

Pastikan tidak ada trailing slash di `apiBaseUrl` dan leading slash konsisten di `apiPath` untuk menghindari double-slash atau missing-slash.

## Tip Pemilihan `port`

Nilai `port` hanya memengaruhi HTTP server statis yang dijalankan oleh `app-start.bat` (memakai `npx serve`). Properti ini **tidak** memengaruhi `apiBaseUrl` atau koneksi ke backend API.

Jika port sudah dipakai proses lain, `npx serve` akan gagal dengan error `EADDRINUSE`. Solusinya: ubah `port`, generate ulang aplikasi, lalu jalankan kembali script start.

## Tip Pemilihan `plugin`

Pilih plugin sesuai kebutuhan aplikasi:

| Plugin | Cocok untuk |
|--------|-------------|
| `vanilla-js-basic` | Prototyping, proof-of-concept, aplikasi internal sederhana tanpa auth |
| `vanilla-js-auth` | Aplikasi produksi yang membutuhkan autentikasi JWT, sidebar navigation, dan permission control |

Plugin lain dapat di-install secara eksternal dan dipakai dengan menulis ID-nya di `plugin`. Jalankan `rfd plugins list` untuk melihat plugin yang ter-install.

---

← [`payload-envelope.md`](./payload-envelope.md) | [Selanjutnya: `page-anatomy.md`](./page-anatomy.md) →
