# `test generate`

> Generate file integration test (Jest + Supertest) untuk endpoint yang sudah ada.

## Pattern

```
npx restforge test generate --project=<NAME> --endpoint=<NAME> [options]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project target |
| `--endpoint <NAME>` | Ya | - | Nama endpoint yang akan di-test |
| `--port <N>` | Tidak | `3000` | Port server saat test dijalankan |
| `--init` | Tidak | `false` | Inisialisasi konfigurasi Jest jika belum ada |

## Contoh

```bat
:: Generate test pertama kali (sekalian init Jest)
npx restforge test generate --project=my-app --endpoint=users --init

:: Generate test tambahan
npx restforge test generate --project=my-app --endpoint=orders --port=3032
```

---

**Lihat juga**: [`test/`](./) · [`commands/`](../) · [`README`](../../../README.md)
