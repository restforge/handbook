# `processor create`

> Generate processor baru dari file payload JSON.

## Pattern

```
npx restforge processor create --project=<NAME> --name=<NAME> --payload=<FILE> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project target |
| `--name <NAME>` | - | Nama processor |
| `--payload <FILE>` | - | Path file payload JSON |

## Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--database <TYPE>` | auto-detect | Tipe database (`postgres`, `mysql`, `oracle`, `sqlite`) |
| `--force` | `false` | Archive lalu timpa file implementasi processor existing (lihat section Perilaku Re-run) |
| `--skip-sql-validation` | `false` | Lewati validasi keyword SQL |

## Contoh

```bat
npx restforge processor create --project=my-app --name=order-validator --payload=order-validator.json
```

## Perilaku Re-run

Menjalankan ulang command pada processor yang sudah ada berperilaku sebagai berikut:

| Kondisi | Endpoint router & metadata | File implementasi processor |
|---------|---------------------------|------------------------------|
| Tanpa `--force` | Selalu di-regenerate | Di-SKIP dengan pesan `custom code preserved`; kode custom aman |
| Dengan `--force` | Selalu di-regenerate | Di-archive terlebih dahulu, lalu ditimpa scaffold hasil generate |

Kontrak ini menjadi kunci workflow custom processor (lihat section "Processor Tanpa SQL" di [catalogs/rdf/processor.md](../../../catalogs/rdf/processor.md)): re-run tanpa `--force` aman dilakukan untuk memperbarui routing tanpa kehilangan business logic yang ditulis manual. Sebaliknya, `--force` menimpa file implementasi sehingga kode custom hanya tersisa di archive.

---

**Lihat juga**: [`processor/`](./) · [`commands/`](../) · [`README`](../../../README.md)
