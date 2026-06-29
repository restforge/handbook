# Resource `data` (Data Pull / Push)

> Export isi tabel database ke file lokal (`data pull`) dan memuatnya kembali ke
> database (`data push`), dengan format file yang **agnostic** lintas dialect.

Resource `data` menyediakan dua verb untuk memindahkan baris data tabel antara
database dan file lokal:

| Verb | Tujuan | Halaman |
|------|--------|---------|
| `pull` | Export baris tabel ke file envelope JSON: satu tabel (`--table`), per schema (`--schema`), atau seluruhnya (`--all-schemas`) | [`pull.md`](./pull.md) |
| `push` | Muat file envelope kembali ke tabel (append-only): satu tabel (`--table`), per schema (`--schema`), atau seluruhnya (`--all-schemas`) | [`push.md`](./push.md) |

```
npx restforge data pull --table=visitors
npx restforge data pull --schema=sales
npx restforge data pull --all-schemas
npx restforge data push --table=visitors
npx restforge data push --schema=sales
npx restforge data push --all-schemas
```

Metadata tabel (daftar kolom, tipe, primary key, namespace) bersumber **murni dari
SDF (Schema Definition File)**, bukan introspeksi database. Akibatnya kedua command
hanya bisa beroperasi pada tabel yang sudah terdeklarasi di SDF.

## Prasyarat SDF

- Tabel target **wajib terdaftar** di SDF yang ditunjuk `--schema-path` (default folder `schema/`).
  Tabel yang tidak terdaftar ditolak sebelum koneksi database dibuka (exit `2`).
- Daftar kolom, tipe canonical, nullable, dan primary key seluruhnya dibaca dari SDF IR.
  Kolom yang tidak dideklarasikan di SDF tidak ikut di-pull dan ditolak saat push.
- Karena SDF hanya menyimpan 10 tipe canonical (`string`, `text`, `integer`, `bigint`,
  `decimal`, `boolean`, `date`, `timestamp`, `uuid`, `json`), kolom binary mustahil
  muncul (eksklusi by-omission). Tidak ada introspeksi database untuk mendeteksi tipe.

## Resolusi Config

Kedua command memerlukan config database (`.env`). Urutan resolusi:

1. `--config=<file>` eksplisit → prioritas tertinggi.
2. Tanpa `--config` → fallback ke default dari `.restforge/defaults.json`
   (ditetapkan via `npx restforge config set-default --config=<file>`). Saat fallback
   aktif, warning dicetak ke stderr.
3. Tanpa `--config` dan tanpa default → error exit `2` dengan tip untuk menjalankan
   `config set-default`.

## Nama File Output

Nama file diturunkan **murni dari schema SDF** tabel, bukan dari flag `--table` mentah.
Tabel ber-schema ditulis ke subfolder bernama schema-nya (layout bersarang), sedangkan
tabel tanpa schema ditulis langsung ke root folder storage:

| Tabel di SDF | Nama file |
|---|---|
| Tanpa schema (`visitors`) | `visitors.json` |
| Ber-schema (`sales.order_item`) | `sales/order_item.json` |

Layout bersarang ini ditentukan murni oleh schema yang dideklarasikan di SDF, sehingga
berlaku untuk semua flag. Pemanggilan `--table=sales.order_item` tetap menghasilkan
`sales/order_item.json` (bukan `sales.order_item.json` ber-titik). Aturan penamaan
**identik** untuk pull dan push, sehingga file hasil pull dapat langsung di-push tanpa
rename. Folder default: `data-storage/` (override dengan `--storage-path`, berlaku untuk
pull dan push).

## Format Envelope

File berformat JSON dengan urutan key tetap: `format_version`, `schema`, `table`,
`source_dialect`, `columns`, `rows`.

```json
{
  "format_version": "1.1",
  "schema": null,
  "table": "visitors",
  "source_dialect": "postgresql",
  "columns": [
    { "name": "id",         "type": "integer",   "nullable": false, "primary_key": true },
    { "name": "name",       "type": "string",    "nullable": false, "primary_key": false },
    { "name": "is_active",  "type": "boolean",   "nullable": true,  "primary_key": false },
    { "name": "metadata",   "type": "json",      "nullable": true,  "primary_key": false },
    { "name": "created_at", "type": "timestamp", "nullable": true,  "primary_key": false }
  ],
  "rows": [
    { "id": "1", "name": "Andi", "is_active": "true",  "metadata": { "tier": "gold" }, "created_at": "2026-06-04T08:30:00" },
    { "id": "2", "name": "Budi", "is_active": "false", "metadata": null,               "created_at": "2026-06-04T09:15:00" }
  ]
}
```

| Field | Keterangan |
|-------|-----------|
| `format_version` | Versi format envelope (saat ini `"1.1"`) |
| `schema` | Nama schema tabel (= `schemaName` SDF; `null` bila tanpa schema). Ditambahkan di format `1.1` |
| `table` | Nama tabel tanpa schema (unqualified) |
| `source_dialect` | Dialect database sumber saat pull (`postgresql`/`mysql`/`oracle`/`sqlite`) |
| `columns` | Daftar kolom whitelist SDF: `{ name, type, nullable, primary_key }` |
| `rows` | Array baris; tiap baris adalah object `{ <kolom>: <nilai> }` |

File hasil pull selalu ditulis pada format `1.1`. Untuk back-compat, `data push` tetap
menerima envelope `1.0` (tanpa field `schema`) maupun `1.1`; envelope `1.0` diperlakukan
seakan `schema` bernilai `null`.

### Representasi Nilai

Mayoritas nilai disimpan sebagai **text** agar portable lintas dialect. Kolom `json`
adalah pengecualian (disimpan sebagai nilai JSON nested asli, bukan string ter-escape).

| Tipe canonical | Representasi di envelope | Contoh |
|---|---|---|
| `string`, `text`, `uuid` | string | `"Andi"` |
| `integer`, `bigint`, `decimal` | string (presisi terjaga, termasuk di luar batas Number) | `"42"`, `"9007199254740993"` |
| `boolean` | string `"true"` / `"false"` | `"true"` |
| `date` | string `YYYY-MM-DD` | `"2026-06-04"` |
| `timestamp` | string ISO-8601 (zona/offset dipertahankan bila ada) | `"2026-06-04T08:30:00"` |
| `json` | nilai JSON nested asli (object/array) | `{ "tier": "gold" }` |
| NULL | `null` | `null` |

## Penanganan Lintas-Dialect

Format envelope dirancang agnostic: satu file hasil pull dari dialect tertentu dapat
di-push ke dialect berbeda. Konversi spesifik per dialect (mis. boolean disimpan
sebagai VARCHAR `'true'`/`'false'` pada non-PostgreSQL, JSON sebagai TEXT/CLOB pada
SQLite/Oracle) ditangani secara otomatis saat push.

| Dialect | Status round-trip |
|---------|-------------------|
| PostgreSQL | Didukung dan terverifikasi |
| MySQL | Didukung dan terverifikasi |
| SQLite | Didukung dan terverifikasi |
| Oracle | Belum didukung penuh pada versi ini |

Round-trip lintas dialect antara PostgreSQL, MySQL, dan SQLite terverifikasi pada
seluruh pasangan arah.

### Semantik `--schema` per Dialect

Flag `--schema` adalah **filter scope** atas tabel SDF, bukan penentu namespace. Makna
schema sebagai namespace berbeda per dialect:

| Dialect | Makna schema |
|---|---|
| PostgreSQL | Namespace nyata (mis. `public`, `sales`) |
| MySQL | Schema setara database |
| SQLite | Hanya tersedia schema tunggal `main` |

Layout file bersarang (`<schema>/<table>.json`) ditentukan murni dari schema yang
dideklarasikan di SDF dan berlaku pada level file di semua dialect. Verifikasi round-trip
DB-real untuk layout bersarang dilakukan pada PostgreSQL.

## Exit Code

Semantik exit code konsisten untuk integrasi scripting dan CI/CD:

| Exit Code | Kategori | Kondisi umum |
|-----------|----------|--------------|
| `0` | Sukses | Operasi selesai |
| `1` | Operational | File output sudah ada (pull tanpa `--force`), file input tidak ada (push), envelope invalid, column mismatch vs SDF, gagal baca SDF |
| `2` | Usage / precondition | `--config` wajib namun tidak ada, bukan tepat satu dari `--table` / `--schema` / `--all-schemas` (pull & push), `--schema` tidak cocok dengan tabel manapun (0-match), tabel tidak terdaftar / ambigu di SDF, `--format` tidak didukung (pull) |
| `3` | Database error | Gagal koneksi atau eksekusi query (termasuk konflik primary key saat append push) |

Catatan edge case: `--all-schemas` pada SDF yang tidak memiliki tabel sama sekali
menghasilkan 0 tabel ter-proses dan exit `0` (benign, bukan error). Sebaliknya `--schema`
yang tidak cocok dengan tabel manapun adalah usage error (exit `2`).

Detail mapping per command ada di [`pull.md`](./pull.md) dan [`push.md`](./push.md).

---

**Lihat juga**: [`pull.md`](./pull.md) · [`push.md`](./push.md) · [`conventions`](../conventions.md) · [`catalogs/sdf/`](../../../catalogs/sdf/) · [`README`](../../README.md)
