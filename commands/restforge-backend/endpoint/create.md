# `endpoint create`

> Generate endpoint REST baru dari file payload JSON. Jika project belum ada, project akan dibuat sebagai bagian dari proses.

## Pattern

```
npx restforge endpoint create --project=<NAME> --name=<NAME> --payload=<FILE> --config=<ENV> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project target |
| `--name <NAME>` | - | Nama endpoint yang akan dibuat |
| `--payload <FILE>` | - | Path atau nama file payload JSON |
| `--config <ENV>` | - | File config database (.env) untuk validasi schema payload-vs-database |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--database <TYPE>` | auto-detect | Tipe database (`postgres`, `mysql`, `oracle`, `sqlite`) |
| `--skip-schema-check` | `false` | Lewati validasi schema database. Escape hatch untuk kondisi database offline atau maintenance. Shape RDF tetap divalidasi |
| `--force` | `false` | Timpa file yang sudah ada di output directory |
| `--create-examples` | `true` | Generate example files (curl, Postman, Insomnia) |
| `--skip-sql-validation` | `false` | Lewati validasi keyword SQL |
| `--no-audit-migration` | `false` | Lewati eksekusi audit table migration saat payload pakai `fieldPolicy` strategy `"audit"` (file SQL tetap ditulis ke `migrations/audit/`). Berguna untuk CI/CD pipeline atau production deployment yang ingin review DDL dulu sebelum eksekusi |
| `--verbose` | `false` | Tampilkan output verbose untuk debugging |

## Mekanisme Alternatif (Alternative Mechanisms)

Pada kondisi default, `--config` wajib disediakan setiap kali command
dijalankan. Tersedia dua mekanisme **opt-in** untuk mengubah behavior
tersebut:

### Pre-setup via `config set-default`

User dapat menyetel file config default per working directory sehingga
tidak perlu mengetik `--config` berulang kali:

```
npx restforge config set-default --config=db.env
```

Setelah perintah ini, command `endpoint create` akan fallback ke
`.restforge/defaults.json` bila `--config` tidak disediakan eksplisit.
Warning di-print ke stderr: `[restforge] Using default config: db.env`.

### Bypass Schema Validation via `--skip-schema-check`

Untuk kondisi khusus seperti database offline, maintenance, atau
tooling tanpa akses DB, validasi schema dapat di-bypass:

```
npx restforge endpoint create ... --skip-schema-check
```

Saat flag ini aktif, validasi schema database dilewati sepenuhnya. Flag
`--config` boleh kosong dalam mode ini karena validator early-return
sebelum membaca config.

## Validasi Schema Database

Command `endpoint create` melakukan cross-check antara
payload JSON dan struktur tabel database aktual sebelum codegen dimulai.
Validasi mencakup:

- Setiap kolom di `fieldName` harus ada di tabel database
- Kolom audit (`created_at`, `created_by`, `updated_at`, `updated_by`) harus
  ada di database bila `auditColumns` aktif (default behavior)
- Kolom database yang tidak ada di payload dilaporkan sebagai diff

Bila ditemukan drift, command exit dengan kode `1` dan tidak menulis file
output apa pun. File yang sudah ada tidak di-archive (tidak ada rotate `*.archive.NNN`).

Detail aturan validasi lihat [`catalogs/rdf/validation-rules.md`](../../../catalogs/rdf/validation-rules.md).

## Exit Code

| Code | Arti |
|------|------|
| `0` | Sukses |
| `1` | Schema drift terdeteksi |
| `2` | Usage error (mis. flag `--config` tidak disediakan) |
| `3` | Connection error ke database |

## Contoh (Examples)

```bat
:: Generate endpoint dengan validasi schema (mode default)
npx restforge endpoint create --project=my-app --name=users --payload=users.json --config=db.env

:: Generate dengan force overwrite (example files dibuat by default)
npx restforge endpoint create --project=my-app --name=orders --payload=orders.json --config=db.env --force

:: Generate tanpa example files
npx restforge endpoint create --project=my-app --name=orders --payload=orders.json --config=db.env --create-examples=false

:: Generate endpoint saat database offline (escape hatch)
npx restforge endpoint create --project=my-app --name=visitors --payload=visitors.json --skip-schema-check

:: Generate endpoint dengan fieldPolicy audit, tulis DDL audit table ke file (skip eksekusi)
npx restforge endpoint create --project=my-app --name=protected_item --payload=protected_item.json --config=db.env --no-audit-migration
```

## Resolusi Drift

Bila command exit dengan kode `1` karena drift, opsi resolusi:

1. Re-sync payload dari database aktual (resolver kanonik untuk drift,
   termasuk audit columns):
   ```
   npx restforge payload sync --table=<table> --config=<ENV>
   ```
2. Tambah kolom missing ke schema database (terutama kolom audit) bila
   memang seharusnya ada
3. Tetapkan `"auditColumns": false` di payload secara manual bila ingin override
   tanpa menjalankan sync

`payload sync` resolve `auditColumns` secara otomatis:
bila tabel tidak punya kolom audit standar, command akan menetapkan
`"auditColumns": false` di payload setelah sync. Detail lihat
[`commands/restforge-backend/payload/sync.md`](../payload/sync.md#resolusi-audit-columns-audit-columns-resolution).

---

**Lihat juga**: [`endpoint/`](./) · [`commands/`](../) · [`README`](../../../README.md)
