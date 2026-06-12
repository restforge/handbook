# `serve`

> Menjalankan runtime HTTP server untuk project RESTForge yang sudah di-generate.

## Pattern

```
npx restforge serve --project=<NAME> [options]
```

## Flag Wajib (Required)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project yang akan dijalankan |

## Flag Opsional (Optional)

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--port <N>` | `3000` | Port HTTP server (range 1-65535) |
| `--server-address <HOST>` | `127.0.0.1` | Alamat bind server (override via env `SERVER_ADDRESS`) |
| `--config <FILE>` | `db-connection.env` | Nama file config database yang dipakai project |
| `--key <FILE>` | - | Path file yang berisi API key untuk autentikasi |
| `--license <KEY>` | - | License key inline (format `XXXX-XXXX-XXXX-XXXX`) |
| `--license-server <URL>` | auto | Endpoint custom license server |
| `--cluster` | `false` | Aktifkan cluster mode (auto-detect jumlah CPU) |
| `--workers <N>` | CPU count | Jumlah worker process (implicit mengaktifkan cluster mode) |
| `--watch` | `false` | Aktifkan file watcher untuk auto-restart saat config berubah |

## Contoh (Examples)

```bat
:: Start basic
npx restforge serve --project=my-project

:: Start dengan port custom dan cluster
npx restforge serve --project=my-project --port=8080 --cluster --workers=4

:: Start dengan license inline dan custom config
npx restforge serve --project=my-project --config=production.env --license=AAAA-BBBB-CCCC-DDDD
```

---

**Lihat juga**: [`runtime/`](./) · [`commands/`](../) · [`README`](../../../README.md)
