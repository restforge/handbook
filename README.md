# RESTForge

> Framework definition-first untuk pengembangan aplikasi web full-stack. Dari struktur database sampai UI siap produksi.

[![npm version](https://img.shields.io/npm/v/@restforgejs/platform.svg)](https://www.npmjs.com/package/@restforgejs/platform)
[![License](https://img.shields.io/badge/license-CC--BY--4.0-blue.svg)](./LICENSE)

---

## Apa Itu RESTForge

RESTForge adalah framework definition-first yang menghasilkan aplikasi full-stack siap produksi dari **tiga jenis file definisi**. Setiap file berperan sebagai rujukan tunggal untuk satu fase pembuatan aplikasi: struktur database, REST API backend, dan UI frontend.

### Tiga Definition Format

| Format | File | Tujuan | Generator |
|--------|------|--------|-----------|
| **SDF** — Schema Definition File | `schema/<table>.js` | Logical schema database | `restforge schema migrate` |
| **RDF** — Resource Definition File | `payload/<resource>.json` | Endpoint REST API | `restforge endpoint create` |
| **UDF** — UI Definition File | `ui-payload/<page>.json` | Halaman frontend | RESTForge Designer |

Ketiga file dapat didefinisikan secara independen, tetapi saling melengkapi:

- RDF dapat di-generate dari struktur database (via `restforge payload generate`)
- UDF dapat dikonversi dari RDF (via Designer)

Konsistensi nama field terjaga karena setiap fase merujuk pada sumber sebelumnya.

---

## Alur Pengembangan

![RESTForge Development Flow](https://pub-6096676046da426597041567e928006f.r2.dev/handbook/development-flow.png)

---

## Quickstart

### Instalasi

Setiap project RESTForge menempati satu folder tersendiri. Folder tersebut menjadi root project yang memuat `node_modules`, `config/`, `schema/`, dan `payload/`. Karena instalasi bersifat lokal pada folder (bukan global), project baru selalu berarti folder baru.

Buat folder project, masuk ke dalamnya, lalu install:

```bash
mkdir my-project
cd my-project
npm install @restforgejs/platform
```

Perintah `npm install` memasang RESTForge ke dalam `node_modules` milik folder tersebut. Seluruh perintah `npx restforge` berikutnya, termasuk `init`, dijalankan dari dalam folder project ini.

### Verifikasi Instalasi

Pastikan RESTForge terpasang dengan benar sebelum melanjutkan:

```bash
npx restforge --version
```

Output menampilkan versi platform, versi Node.js, dan informasi platform:

```
RESTForge Platform v5.1.7
Node.js v22.21.1
Platform: win32 (x64)
```

### Inisialisasi Project

Project perlu diinisialisasi terlebih dahulu agar file konfigurasi
`config/db-connection.env` tersedia. Tanpa langkah ini, perintah RESTForge yang
membutuhkan koneksi database (mis. `schema migrate`) akan error karena file config
belum ada.

```bash
npx restforge init
```

Perintah ini bersifat interaktif dan meminta beberapa nilai konfigurasi (tekan Enter
untuk menerima nilai default):

```
RESTForge - Init
====================

Enter configuration (press Enter to accept the default value):

  LICENSE (XXXX-XXXX-XXXX-XXXX): ABCD-1234-5678-EFGH

  Supported DB_TYPE: postgresql, mysql, oracle, sqlite
  DB_TYPE (postgresql):postgresql
  DB_HOST (127.0.0.1): 127.0.0.1
  DB_PORT (5432): 5432
  DB_USER (postgres): postgres
  DB_PASSWORD (your_password_here): your_password
  DB_NAME (your_database_name): dbdemo

  Config file name (db-connection.env):
```

Setelah konfirmasi, RESTForge menulis file config beserta sample files:

```
  Created: config\db-connection.env

Done! Sample files generated successfully.
```

Bila kredensial database perlu disesuaikan, edit langsung file `config/db-connection.env`.

---

## Onboarding via Playbook

Setelah setup berhasil, langkah berikutnya yang disarankan adalah mengikuti **RESTForge Playbook**, yaitu repository berisi contoh project lengkap untuk onboarding sekaligus ujicoba aplikasi secara end-to-end.

```bash
git clone https://github.com/restforge/playbook.git
```

Playbook memandu dari definisi SDF dan RDF hingga REST API yang berjalan, sehingga keseluruhan alur pengembangan dapat langsung dipraktikkan pada contoh nyata. Ikuti instruksi pada [github.com/restforge/playbook](https://github.com/restforge/playbook) untuk menjalankan skenario onboarding atau ujicoba aplikasi.

---

## Catatan Arsitektur REST API

RESTForge memakai istilah **REST API** dalam pengertian luas sebagaimana lazim di industri, yaitu API berbasis HTTP yang resource-oriented dengan format JSON. Secara teknis, gaya arsitektur yang dihasilkan adalah **action-based (RPC-over-HTTP)**, bukan RESTful murni menurut Richardson Maturity Model (RMM) Level 2 ke atas.

Karakteristik konkret endpoint yang di-generate:

- Endpoint mengikuti pola `POST /{resource}/{action}`, misal `POST /category/create` dan `POST /category/update`
- Action verb berada di dalam URL, tidak di-encode melalui HTTP method
- Mayoritas operasi memakai HTTP method POST, termasuk operasi baca seperti `/read` dan `/first`
- HTTP status code tetap dipakai secara semantik (`400` untuk validasi, `409` untuk konflik)

Berdasarkan pola tersebut, RESTForge berada pada **RMM Level 1** (resource-oriented URI dengan action di path), bukan Level 2 yang memetakan operasi ke HTTP method (`GET`, `POST`, `PUT`, `DELETE`). Istilah REST API dipertahankan agar akrab bagi pembaca, tetapi perlu dipahami sebagai HTTP API bergaya action-based, bukan RESTful sesuai definisi formal Roy Fielding (2000).

---

## Navigasi Repo

| Folder | Isi |
|--------|-----|
| [`commands/`](./commands/) | Reference perintah CLI yang berlaku, dikelompokkan per resource |
| [`catalogs/sdf/`](./catalogs/sdf/) | Schema Definition File: field types, constraints, relations, dialect support |
| [`catalogs/rdf/`](./catalogs/rdf/) | Resource Definition File: field validation, query declarative, audit columns |
| [`catalogs/udf/`](./catalogs/udf/) | UI Definition File: page structure, component types, design system |
| [`api-spec/`](./api-spec/) | Spesifikasi REST API endpoint hasil generate: pola URL, envelope response, kode status HTTP, WHERE clause, sort columns |
| [`examples/`](./examples/) | File contoh SDF, RDF, UDF yang siap di-copy-paste |

---

## Konvensi CLI

Seluruh perintah RESTForge mengikuti pattern resource-first dengan flag-only argument:

```
npx restforge <resource> <verb> [options]
```

- `<resource>` dan `<verb>` adalah positional wajib (mis. `schema validate`)
- Seluruh argument lain via flag bernama (mis. `--path=<PATH>`, `--config=<FILE>`)
- Tidak ada positional argument tambahan setelah verb

Detail lengkap di [`commands/README.md`](./commands/README.md).

---

## Fitur Inti

| Aspek | Mekanisme |
|-------|-----------|
| **Multi-Database** | PostgreSQL, MySQL, Oracle, SQLite dengan template generator per dialect |
| **High Availability** | Cluster mode, PM2, zero-downtime deploy, graceful shutdown, multi-server |
| **Horizontal Scaling** | Runtime stateless, shared state via Redis |
| **Reliability Primitives** | Cache, distributed lock, idempotency, rate limit, ID generator, fieldPolicy |
| **Security Layer** | Request scope (row-level security), Helmet headers, API key authentication |
| **Real-time Sync** | Live Sync sebagai dedicated WebSocket process berbasis Redis |
| **Background Processing** | Job scheduler berbasis BullMQ, export/import Excel async |

---

## Lifecycle Generated Code

Generated code RESTForge **bebas di-edit** dengan tooling JavaScript biasa. File hasil generate adalah aset tim yang disimpan di repository, dikelola versinya dengan Git.

Saat tim perlu menyesuaikan perilaku:

| Jalur | Pendekatan | Trade-off |
|-------|-----------|-----------|
| **Edit source langsung** | Modifikasi file hasil generate | File lama otomatis di-archive sebagai `*.archive.NNN` saat regenerate |
| **Revisi definition + regenerate** | Update SDF/RDF/UDF, lalu generate ulang | Perubahan terjaga di setiap regenerate, namun terbatas pada apa yang dapat dideklarasikan |

Untuk logic custom yang tidak dapat dideklarasikan: gunakan **titik ekstensi** (component handler, custom processor, server plugin) yang tidak tertimpa saat regenerate.

**Rekomendasi**: prioritaskan **definition-first** untuk konsistensi dan portabilitas.

---

## Kontribusi

Repo ini berisi dokumentasi RESTForge. Source code engine berada di repo terpisah (private). Untuk perbaikan dokumentasi:

1. Fork repo ini
2. Edit file relevan
3. Submit pull request dengan deskripsi singkat alasan perubahan

---

## Lisensi

Konten dokumentasi di repo ini dilisensikan di bawah [Creative Commons Attribution 4.0 International (CC-BY-4.0)](./LICENSE). Bebas digunakan, dimodifikasi, dan didistribusikan dengan menyertakan atribusi ke RESTForge.

---

*Dokumentasi dipelihara oleh RESTForge Development Team.*
