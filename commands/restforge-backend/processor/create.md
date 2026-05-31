# `processor create`

> Generate processor baru dari file payload JSON.

## Pattern

```
npx restforge processor create --project=<NAME> --name=<NAME> --payload=<FILE> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project target |
| `--name <NAME>` | - | Nama processor |
| `--payload <FILE>` | - | Path file payload JSON |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--database <TYPE>` | auto-detect | Tipe database (`postgres`, `mysql`, `oracle`, `sqlite`) |
| `--force` | `false` | Timpa file existing |
| `--skip-sql-validation` | `false` | Lewati validasi keyword SQL |

## Contoh (Examples)

```bat
npx restforge processor create --project=my-app --name=order-validator --payload=order-validator.json
```

---

**Lihat juga**: [`processor/`](./) · [`commands/`](../) · [`README`](../../../README.md)
