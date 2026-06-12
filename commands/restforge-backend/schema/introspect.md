# `schema introspect`

> Reverse-engineer database existing menjadi file schema definition (dbschema-kit factory function).

## Pattern

```
npx restforge schema introspect --config=<FILE> --schema-path=<PATH> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--config <FILE>` | - | File config database |
| `--schema-path <PATH>` | - | Folder atau file output |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--table <NAME>` | semua | Hanya satu tabel (unqualified atau qualified) |
| `--schema <LIST>` | default schema | Filter schema (comma-separated) |
| `--all-schemas` | `false` | Auto-detect seluruh user schemas |
| `--force` | `false` | Timpa file existing |
| `--dry-run` | `false` | Print ke stdout tanpa menulis file |

## Contoh (Examples)

```bat
:: Introspect seluruh tabel
npx restforge schema introspect --config=db.env --schema-path=./schema

:: Introspect tabel spesifik dengan dry-run
npx restforge schema introspect --config=db.env --schema-path=./schema --table=users --dry-run

:: Introspect seluruh user schemas
npx restforge schema introspect --config=db.env --schema-path=./schema --all-schemas --force
```

## Kondisi Blokir Soft-Delete (Strict)

Pada PostgreSQL, tabel yang memiliki kolom bernama soft-delete (`is_deleted`/`deleted_at`/`deleted_by`) divalidasi terhadap [kontrak soft-delete](../../../catalogs/sdf/soft-delete.md). Tabel yang valid (tiga kolom lengkap, tipe sesuai, CHECK konsistensi ada) menghasilkan SDF dengan blok `softDelete: { enabled: true }`. Tabel yang **tidak** memenuhi kontrak memblokir introspect dengan ERROR dan exit code 1, dalam tiga bentuk:

| Kondisi | Inti pesan |
|---------|-----------|
| Kolom parsial (hanya sebagian dari tiga kolom) | `The table has some soft-delete columns, but the set is incomplete` beserta daftar kolom found/missing |
| Tipe kolom tidak sesuai kontrak | `the soft-delete columns are complete, but their types do not match the contract` beserta daftar found/required per kolom |
| CHECK konsistensi tidak ditemukan | `the consistency CHECK constraint was not found`, disertai SQL `ALTER TABLE ... ADD CONSTRAINT` siap pakai untuk menambahkannya |

Setiap pesan menyertakan opsi mitigasi: melengkapi kontrak (tambah kolom/perbaiki tipe/tambah CHECK) atau menghapus kolom soft-delete dari tabel. Aturan ini menjaga SDF hasil introspect tidak pernah memuat state soft-delete yang setengah jadi.

Jalur `schema diff` dan `schema apply` tidak strict: drift soft-delete dilaporkan apa adanya tanpa memblokir, sehingga tidak ada operasi destruktif yang terpicu dari kondisi ini.

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
