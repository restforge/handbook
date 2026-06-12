# `schema diff`

> Drift detection antara Schema Definition File (SDF) dan struktur tabel actual di database. Bersifat read-only dan bidirectional (only-in-SDF, only-in-DB, mismatched).

## Pattern

```
npx restforge schema diff --path=<PATH> --config=<FILE> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--path <PATH>` | - | Path file atau folder schema |
| `--config <FILE>` | - | File config database |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--table <NAME>` | semua | Diff hanya satu tabel spesifik |
| `--json` | `false` | Output format JSON (default: human-readable plain) |

## Exit Code

| Code | Arti |
|------|------|
| `0` | Tidak ada drift |
| `1` | Drift terdeteksi |
| `2` | Error (config tidak valid, tabel tidak ditemukan, koneksi gagal, dll.) |

## Contoh (Examples)

```bat
:: Diff seluruh model di folder schema
npx restforge schema diff --path=./schema --config=db.env

:: Diff hanya satu tabel
npx restforge schema diff --path=./schema/visitors.js --config=db.env --table=visitors

:: Output JSON untuk consumption CI/CD
npx restforge schema diff --path=./schema --config=db.env --json
```

---

**Lihat juga**: [`schema/`](./) · [`commands/`](../) · [`README`](../../../README.md)
