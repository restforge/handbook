# RESTForge

> Definition-first framework untuk full-stack web application development. Dari struktur database sampai UI siap produksi, tanpa kompromi.

[![npm version](https://img.shields.io/npm/v/@restforgejs/platform.svg)](https://www.npmjs.com/package/@restforgejs/platform)
[![License](https://img.shields.io/badge/license-CC--BY--4.0-blue.svg)](./LICENSE)

---

## Apa Itu RESTForge (What is RESTForge)

RESTForge adalah framework definition-first yang menghasilkan aplikasi full-stack siap produksi dari **tiga jenis file definisi**. Setiap file berperan sebagai source of truth untuk satu fase pembuatan aplikasi: struktur database, REST API backend, dan UI frontend.

### Tiga Definition Format

| Format | File | Tujuan | Generator |
|--------|------|--------|-----------|
| **SDF** — Schema Definition File | `schema/<table>.js` | Logical schema database | `restforge schema migrate` |
| **RDF** — Resource Definition File | `payload/<resource>.json` | Endpoint REST API | `restforge endpoint create` |
| **UDF** — UI Definition File | `ui-payload/<page>.json` | Halaman frontend | RESTForge Designer |

Ketiga file dapat didefinisikan independen, namun saling melengkapi:

- RDF dapat di-generate dari struktur database (via `restforge payload generate`)
- UDF dapat dikonversi dari RDF (via Designer)

Konsistensi nama field terjaga karena setiap fase merujuk pada sumber sebelumnya.

---

## Alur Pengembangan (Development Flow)

![RESTForge Development Flow](https://pub-6096676046da426597041567e928006f.r2.dev/handbook/development-flow.png)

---

## Quickstart

### Instalasi (Installation)

```bash
npm install @restforgejs/platform
```

### Siklus Iterasi (Iteration Cycle)

```bash
# 1. Tulis SDF pada folder schema misal: schema/product.js
# 2. Apply ke database
npx restforge schema migrate --path=./schema --config=config/db-connection.env

# 3. Tulis RDF pada folder payload misal: payload/product.json
# 4. Generate endpoint
npx restforge endpoint create --project=mystore --name=product --payload=product.json --config=db.connection.env

# 5. Jalankan API
npx restforge serve --project=mystore --config=config/db-connection.env
```

REST API siap melayani request di port yang dikonfigurasi.

---

## Catatan Arsitektur REST API (REST API Architecture Note)

RESTForge memakai istilah **REST API** dalam pengertian luas sebagaimana lazim di industri, yaitu API berbasis HTTP yang resource-oriented dengan format JSON. Secara teknis, gaya arsitektur yang dihasilkan adalah **action-based (RPC-over-HTTP)**, bukan RESTful murni menurut Richardson Maturity Model (RMM) Level 2 ke atas.

Karakteristik konkret endpoint yang di-generate:

- Endpoint mengikuti pola `POST /{resource}/{action}`, misal `POST /category/create` dan `POST /category/update`
- Action verb berada di dalam URL, tidak di-encode melalui HTTP method
- Mayoritas operasi memakai HTTP method POST, termasuk operasi baca seperti `/read` dan `/first`
- HTTP status code tetap dipakai secara semantik (`400` untuk validasi, `409` untuk konflik)

Berdasarkan pola tersebut, RESTForge berada pada **RMM Level 1** (resource-oriented URI dengan action di path), bukan Level 2 yang memetakan operasi ke HTTP method (`GET`, `POST`, `PUT`, `DELETE`). Istilah REST API dipertahankan demi familiaritas pembaca, namun perlu dipahami sebagai HTTP API bergaya action-based, bukan RESTful sesuai definisi formal Roy Fielding (2000).

---

## Navigasi Repo (Repository Navigation)

| Folder | Isi |
|--------|-----|
| [`commands/`](./commands/) | Reference perintah CLI yang berlaku, dikelompokkan per resource |
| [`catalogs/sdf/`](./catalogs/sdf/) | Schema Definition File: field types, constraints, relations, dialect support |
| [`catalogs/rdf/`](./catalogs/rdf/) | Resource Definition File: field validation, query declarative, audit columns |
| [`catalogs/udf/`](./catalogs/udf/) | UI Definition File: page structure, component types, design system |
| [`api-spec/`](./api-spec/) | Spesifikasi REST API endpoint hasil generate: pola URL, envelope response, kode status HTTP, WHERE clause, sort columns |
| [`examples/`](./examples/) | File contoh SDF, RDF, UDF yang siap di-copy-paste |

---

## Konvensi CLI (CLI Convention)

Seluruh perintah RESTForge mengikuti pattern resource-first dengan flag-only argument:

```
npx restforge <resource> <verb> [options]
```

- `<resource>` dan `<verb>` adalah positional wajib (mis. `schema validate`)
- Seluruh argument lain via flag bernama (mis. `--path=<PATH>`, `--config=<FILE>`)
- Tidak ada positional argument tambahan setelah verb

Detail lengkap di [`commands/README.md`](./commands/README.md).

---

## Fitur Inti (Core Features)

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

## Lifecycle Generated Code (Generated Code Lifecycle)

Generated code yang dihasilkan RESTForge **bebas di-edit** dengan tooling JavaScript biasa. File hasil generate adalah aset tim yang disimpan di repository, di-version dengan Git.

Saat tim perlu menyesuaikan perilaku:

| Jalur | Pendekatan | Trade-off |
|-------|-----------|-----------|
| **Edit source langsung** | Modifikasi file hasil generate | File lama otomatis di-archive sebagai `*.archive.NNN` saat regenerate |
| **Revisi definition + regenerate** | Update SDF/RDF/UDF, lalu generate ulang | Perubahan ter-preserve di setiap regenerate, namun terbatas pada apa yang dapat dideklarasikan |

Untuk logic custom yang tidak dapat dideklarasikan: gunakan **titik ekstensi** (component handler, custom processor, server plugin) yang tidak ter-overwrite saat regenerate.

**Rekomendasi**: prioritaskan **definition-first** untuk konsistensi dan portabilitas.

---

## Kontribusi (Contributing)

Repo ini berisi dokumentasi RESTForge. Source code engine berada di repo terpisah (private). Untuk perbaikan dokumentasi:

1. Fork repo ini
2. Edit file relevan
3. Submit pull request dengan deskripsi singkat alasan perubahan

---

## Lisensi (License)

Konten dokumentasi di repo ini dilisensikan di bawah [Creative Commons Attribution 4.0 International (CC-BY-4.0)](./LICENSE). Bebas digunakan, dimodifikasi, dan didistribusikan dengan menyertakan atribusi ke RESTForge.

---

*Dokumentasi dipelihara oleh RESTForge Development Team.*
