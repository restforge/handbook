# `schema apply`

> Apply drift dari SDF ke database melalui `ALTER TABLE` incremental. Komplemen `schema diff` (detect only) dan `schema migrate` (full create/drop). Default konservatif: hanya additive (ADD COLUMN/INDEX/UNIQUE). Operasi destruktif memerlukan opt-in eksplisit.

## Pattern

```
npx restforge schema apply --path=<PATH> --config=<FILE> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--path <PATH>` | - | Path file atau folder schema |
| `--config <FILE>` | - | File config database |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--table <NAME>` | semua | Apply hanya satu tabel spesifik |
| `--dry-run` | `false` | Preview ALTER tanpa apply ke database |
| `--allow-drop` | `false` | Opt-in: izinkan DROP COLUMN/INDEX/UNIQUE constraint (destruktif data) |
| `--allow-modify` | `false` | Opt-in: izinkan ALTER COLUMN length/nullable (potential data loss) |

## Exit Code

| Code | Arti |
|------|------|
| `0` | Semua drift applicable diterapkan (atau no drift) |
| `1` | Ada drift yang dilewati karena butuh `--allow-drop` atau `--allow-modify` |
| `2` | Error (config tidak valid, tabel tidak ditemukan, koneksi gagal, apply gagal, dll.) |

## Operasi yang Belum Didukung (Deferred)

Beberapa operasi belum ditangani di MVP karena memerlukan strategi tambahan: ALTER COLUMN type change (perlu data conversion), Primary Key changes (perlu rebuild table), Foreign Key ADD/DROP, default value change, check constraint change, dan ALTER COLUMN di SQLite (perlu rebuild table). Operasi ini dilewati dengan reason `deferred` di output dan tidak menyebabkan exit code 1.

## Contoh (Examples)

```bat
:: Preview ALTER incremental sebelum apply
npx restforge schema apply --path=./schema --config=db.env --dry-run

:: Apply additive drift (default konservatif)
npx restforge schema apply --path=./schema --config=db.env

:: Apply termasuk drop kolom legacy
npx restforge schema apply --path=./schema --config=db.env --allow-drop

:: Apply hanya satu tabel
npx restforge schema apply --path=./schema/visitors.js --config=db.env --table=visitors --dry-run
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
