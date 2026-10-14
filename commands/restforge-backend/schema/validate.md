# `schema validate`

> Memvalidasi file schema definition, baik per model maupun antarmodel. Dengan `--config`, SDF juga dibandingkan dengan struktur tabel di database dan setiap model mendapat verdict.

## Pattern

```
npx restforge schema validate --schema-path=<PATH> [--config=<FILE>] [--table=<NAME>] [--json]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--schema-path <PATH>` | Ya | - | Path file atau folder schema (mis. `./schema` atau `./schema/users.js`) |
| `--config <FILE>` | Tidak | - | File config database. Bila diisi, validasi diperluas ke perbandingan dengan struktur tabel di database. Config default dari `config set-default` tidak dipakai |
| `--table <NAME>` | Tidak | semua | Batasi laporan ke satu tabel. Berlaku di kedua mode |
| `--json` | Tidak | `false` | Tulis hasil dalam format JSON ke stdout. Berlaku di kedua mode |

## Mode

| Mode | Pemicu | Isi pemeriksaan |
|------|--------|-----------------|
| Hanya file | `--config` tidak diisi | Validasi per model dan antarmodel. Database tidak dihubungi |
| File dan database | `--config` diisi | Validasi file, lalu setiap model dibandingkan dengan tabelnya di database |

Mode file dan database hanya aktif bila `--config` ditulis. Project yang sudah punya config default tetap berjalan dalam mode hanya file selama `--config` tidak diisi.

Contoh output mode hanya file:

```
✓ D:\myapp\schema\visitor_categories.js: visitor_categories
✓ D:\myapp\schema\visitors.js: visitors

2 models loaded, 0 errors.
```

Contoh output mode file dan database:

```
[OK]    visitor_categories
[DRIFT] visitors: column category_id only in SDF, index (category_id) only in SDF, foreign key category_id only in SDF

2 models checked, 1 drift.
```

## Verdict

Verdict hanya muncul di mode file dan database.

| Verdict | Arti |
|---------|------|
| `[OK]` | SDF valid dan sesuai dengan struktur tabel di database |
| `[DRIFT]` | SDF valid, tetapi kolom, tipe, index, unique, atau foreign key berbeda dengan database |
| `[ERROR]` | Model tidak bisa dibandingkan dengan database. Kategorinya ditulis di akhir baris |

| Kategori `[ERROR]` | Arti | Tindakan |
|--------------------|------|----------|
| `table-missing` | SDF valid, tetapi tabelnya belum ada di database | Jalankan [`schema migrate`](./migrate.md) atau [`schema apply`](./apply.md) |
| `sdf-invalid` | File SDF gagal validasi, sehingga model ini tidak dibandingkan dengan database | Perbaiki file SDF |

```
[OK]    visitor_categories
[ERROR] visitors: table not found in database (table-missing)
[ERROR] orders: SDF validation failed (sdf-invalid)
        ✗ relation 'customer' targets unknown table 'customers'

3 models checked, 2 errors (1 table-missing, 1 sdf-invalid).
```

Rincian penyimpangan per kolom ditampilkan oleh [`schema diff`](./diff.md). Penyimpangan kolom, index, unique, dan foreign key bisa diterapkan ke database lewat [`schema apply`](./apply.md).

Kesalahan penulisan yang membuat file SDF gagal dimuat, misalnya tipe field salah, tidak menghasilkan verdict. Command langsung berhenti dengan exit code `1`.

## Membatasi Laporan dengan `--table`

Seluruh file di `--schema-path` tetap dimuat dan divalidasi, karena validasi foreign key membutuhkan model tujuan. Yang dibatasi hanya laporannya.

- Di mode hanya file, laporan hanya berisi temuan milik tabel yang diminta. Error di tabel lain tidak memengaruhi exit code.
- Di mode file dan database, hanya tabel yang diminta yang dibaca strukturnya dari database.
- Bila nama tabel tidak ada di SDF, command keluar dengan exit code `2`.

```
> npx restforge schema validate --schema-path=./schema --table=orders
✗ Table 'orders': relation 'customer' targets unknown table 'customers'

40 models loaded, 1 error (filtered to orders).
```

`--schema-path` yang menunjuk satu file tidak bisa menggantikan `--table`. Model lain tidak ikut dimuat, sehingga foreign key ke model tersebut dilaporkan menunjuk tabel yang tidak dikenal.

Ada dua batasan penyaringan:

- File SDF yang gagal dimuat tetap menggagalkan command, walaupun file itu bukan tabel yang diminta.
- Temuan relasi dicatat atas nama tabel pemilik relasi. Bila `orders` punya foreign key yang salah ke `visitors`, temuan muncul pada `--table=orders` dan tidak muncul pada `--table=visitors`, karena file yang perlu diperbaiki adalah `orders`. Temuan relasi melingkar muncul untuk setiap tabel di dalam lingkaran.

## Output JSON

`--json` memakai susunan yang sama di kedua mode. Mode file dan database menambah field `models` dan angka verdict di `summary`.

Mode hanya file:

```json
{
  "schemaVersion": "1.0",
  "mode": "file",
  "summary": { "modelsLoaded": 2, "errorCount": 1, "warningCount": 0 },
  "issues": [
    {
      "severity": "error",
      "code": "relation-unknown-target",
      "message": "Table 'orders': relation 'customer' targets unknown table 'customers'",
      "file": "D:\\myapp\\schema\\orders.js",
      "table": "orders",
      "relation": "customer"
    }
  ]
}
```

Mode file dan database:

```json
{
  "schemaVersion": "1.0",
  "mode": "database",
  "summary": {
    "modelsLoaded": 3, "errorCount": 1, "warningCount": 0,
    "modelsChecked": 3, "ok": 1, "drift": 0, "error": 2
  },
  "issues": [ ... ],
  "models": [
    { "table": "visitor_categories", "verdict": "ok" },
    { "table": "visitors", "verdict": "error", "reason": "table-missing" },
    { "table": "orders", "verdict": "error", "reason": "sdf-invalid" }
  ]
}
```

| Field | Keterangan |
|-------|-----------|
| `mode` | `file` atau `database`. Field `models` hanya ada bila nilainya `database` |
| `summary` | Selalu ada. Di mode database ditambah `modelsChecked`, `ok`, `drift`, dan `error` |
| `issues` | Temuan validasi dengan `severity`, `code`, `message`, dan bila tersedia `file`, `table`, `field`, `relation`. Temuan relasi melingkar menambah `tables` berisi semua tabel dalam lingkaran |
| `models[].verdict` | `ok`, `drift`, atau `error` |
| `models[].reason` | Ada pada verdict `error`, bernilai `table-missing` atau `sdf-invalid` |
| `models[].delta` | Ada pada verdict `drift`. Isinya sama dengan entri `tables[]` pada `schema diff --json` (`fields`, `primaryKey`, `indexes`, `uniques`, `foreignKeys`, dan bila ada `checks` serta `softDelete`) |

`--table` menyaring `issues` dan `models` dengan aturan yang sama seperti output teks, dan angka di `summary` dihitung setelah penyaringan. Bila SDF gagal dimuat atau koneksi database gagal, command berhenti dan menulis pesan error ke stderr dengan exit code bukan nol.

## Exit Code

| Code | Arti |
|------|------|
| `0` | Tidak ada error. Di mode file dan database, semua model `[OK]` |
| `1` | Ada error validasi, atau ada model berstatus `[DRIFT]` atau `[ERROR]` |
| `2` | Config tidak valid, koneksi database gagal, atau nama pada `--table` tidak ada di SDF |

Perbandingan dengan database mendukung PostgreSQL, MySQL, dan Oracle. SQLite belum didukung, sama seperti `schema diff`.

## Contoh

```bat
:: Validasi file saja
npx restforge schema validate --schema-path=./schema

:: Validasi file dan bandingkan dengan database
npx restforge schema validate --schema-path=./schema --config=db.env

:: Laporan satu tabel
npx restforge schema validate --schema-path=./schema --config=db.env --table=visitors

:: Output JSON untuk pipeline CI/CD
npx restforge schema validate --schema-path=./schema --config=db.env --json
```

---

**Lihat juga**: [`schema/`](./) · [`schema diff`](./diff.md) · [`commands/`](../) · [`README`](../../../README.md)
