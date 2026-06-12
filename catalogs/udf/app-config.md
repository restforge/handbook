# Konfigurasi Aplikasi — `appConfig`

> Konfigurasi level aplikasi: nama, plugin target, base URL API, port server statis.

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
            "decimalPlaces": 2
        }
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
| `numberFormat` | object | — | ✗ | Konfigurasi default format angka di seluruh aplikasi |
| `numberFormat.decimalPlaces` | integer | — | ✗ | Jumlah angka di belakang koma. Range valid: 0–20 |

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
| `numberFormat.decimalPlaces` harus integer 0–20 | `"appConfig.numberFormat.decimalPlaces must be an integer between 0 and 20, got <value>"` |

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

Plugin lain dapat di-install eksternal dan dipakai dengan menulis ID-nya di `plugin`. Jalankan `rfd plugins list` untuk melihat plugin yang ter-install.

---

← [`payload-envelope.md`](./payload-envelope.md) | [Selanjutnya: `page-anatomy.md`](./page-anatomy.md) →
