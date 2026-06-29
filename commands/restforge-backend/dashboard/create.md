# `dashboard create`

> Generate dashboard endpoint dari file payload yang berisi multi-widget configuration. Nama dashboard harus diawali prefix `dash-`.

## Pattern

```
npx restforge dashboard create --project=<NAME> --name=<DASH-NAME> --payload=<FILE> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project target |
| `--name <DASH-NAME>` | - | Nama dashboard (wajib diawali `dash-`, mis. `dash-sales`) |
| `--payload <FILE>` | - | Path file payload JSON dashboard |

## Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--database <TYPE>` | `postgres` | Tipe database (`postgres`, `mysql`, `oracle`). SQLite ditolak khusus untuk dashboard. |
| `--force` | `false` | Timpa file yang sudah ada |
| `--skip-sql-validation` | `false` | Lewati validasi keyword SQL |

## Contoh

```bat
npx restforge dashboard create --project=mini-inventory --name=dash-sales --payload=dashboard-sales.json
```

---

**Lihat juga**: [`dashboard/`](./) · [`commands/`](../) · [`README`](../../../README.md)
