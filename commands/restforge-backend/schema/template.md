# `schema template`

> Browse, preview, dan generate template schema dari koleksi RestForge Schema Reference (87 template, 30+ domain aplikasi, 33 bagian kategorisasi). Verb ini adalah wrapper terhadap native binary `sdf-tools.exe` yang di-bundle di folder `bin/` package.

## Pattern

```
npx restforge schema template [options]
```

Tanpa argumen, command menampilkan seluruh template dalam format tabel ASCII.

## Flag

### Filter

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--domain <CSV>` | Tidak | semua | Filter domain aplikasi (mis. `erp`, `erp,finance,hr`) |
| `--table <PATTERN>` | Tidak | semua | Filter nama tabel, wildcard glob didukung (`sales*`, `*_invoice`) |
| `--category <CODE>` | Tidak | semua | Filter category: `master-data` atau `transactional` |
| `--pattern <CODE>` | Tidak | semua | Filter pattern: `single-table` atau `master-detail` |
| `--section <CODE>` | Tidak | semua | Filter section code (lihat `--list-sections`) |
| `--has-sdf` | Tidak | `false` | Hanya template yang sudah memiliki versi SDF |
| `--no-sdf` | Tidak | `false` | Hanya template yang belum memiliki SDF (gap analysis) |

### Display

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--show` | Tidak | `false` | Cetak schema template (perlu `--table=<nama_spesifik>`) |
| `--example` | Tidak | `false` | Sertakan section CONTOH DATA (kombinasi dengan `--show`) |
| `--lang <CODE>` | Tidak | `sdf` | Format schema: `sdf` atau `sql` |
| `--format <CODE>` | Tidak | `table` | Format daftar: `table`, `plain`, atau `json` |

### Generate

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--generate` | Tidak | `false` | Tulis template ke filesystem (perlu `--table` dan `--schema-path`) |
| `--schema-path <DEST>` | Tidak | - | Path destination (direktori atau file `.sdf`/`.js`/`.sql`) |
| `--force` | Tidak | `false` | Overwrite file destination yang sudah ada |

### Utility

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--list-domains` | Tidak | `false` | List semua domain aplikasi yang tersedia |
| `--list-categories` | Tidak | `false` | List semua category template |
| `--list-sections` | Tidak | `false` | List semua section beserta category-nya |
| `--stats` | Tidak | `false` | Tampilkan statistik koleksi |

## Contoh

```bat
:: Browse seluruh koleksi
npx restforge schema template

:: Filter berdasarkan domain
npx restforge schema template --domain=erp
npx restforge schema template --domain=erp,inventory --category=master-data

:: Preview schema satu tabel (SDF default)
npx restforge schema template --table=sales_order --show

:: Preview format SQL DDL
npx restforge schema template --table=sales_order --show --lang=sql

:: Preview dengan section CONTOH DATA
npx restforge schema template --table=product --show --example

:: Generate ke filesystem
npx restforge schema template --table=sales_order --generate --schema-path=./schema --lang=sdf
npx restforge schema template --table=category --generate --schema-path=./schema --lang=sql --force

:: Statistik dan eksplorasi
npx restforge schema template --stats
npx restforge schema template --list-domains
npx restforge schema template --list-sections

:: Integrasi dengan tools lain
npx restforge schema template --domain=hr --format=plain
npx restforge schema template --format=json > erp_templates.json
```

## Platform

Saat ini binary `sdf-tools.exe` hanya tersedia untuk Windows. Pemanggilan di Linux atau macOS akan return error eksplisit dengan exit code 3.

## Catatan Penting

- `--show` dan `--generate` tidak mendukung wildcard pada `--table` (harus nama tabel spesifik)
- `--example` hanya valid bersama `--show`
- Master-detail template (mis. `sales_order` + `sales_order_item`) menghasilkan 2 file SDF dengan `--generate`
- Default format `--lang` adalah `sdf` (JavaScript Schema Definition File), bukan SQL
- Detail lengkap fitur binary tersedia di [`USER-GUIDE.md`](../../../../packages/platform/tools/template-table/sdf-tools/USER-GUIDE.md)

## Exit Code

| Code | Arti |
|------|------|
| `0` | Operasi sukses |
| `1` | Error eksekusi binary (filter invalid, file conflict, dll) |
| `2` | Error parsing argumen CLI |
| `3` | Binary tidak ditemukan atau platform tidak didukung |

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
