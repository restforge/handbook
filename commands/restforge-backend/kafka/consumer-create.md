# `kafka consumer-create`

> Generate Kafka consumer code dari file payload JSON. Output berupa source code di `src/consumers/<project>/<name>/` yang nantinya dijalankan oleh binary internal `restforge-consumer`.

## Pattern

```
npx restforge kafka consumer-create --project=<NAME> --name=<NAME> --payload=<FILE> [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project target |
| `--name <NAME>` | Ya | - | Nama consumer yang akan dibuat |
| `--payload <FILE>` | Ya | - | Path file payload JSON consumer |
| `--force` | Tidak | `false` | Timpa file yang sudah ada |

## Contoh (Examples)

```bat
npx restforge kafka consumer-create --project=my-app --name=order-events --payload=order-events.json
```

---

**Lihat juga**: [`kafka/`](./) · [`commands/`](../) · [`internal-binary`](../internal-binary.md) · [`README`](../../../README.md)
