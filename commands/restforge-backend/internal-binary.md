# Internal Binary

> Binary tambahan di luar pattern resource-first. Ditujukan untuk maintainer atau advanced user.

Binary pada bagian ini belum mengikuti pattern resource-first dan berdiri sendiri tanpa prefix `npx restforge`. Pemanggilan menggunakan binary independen (`restforge-consumer`, `restforge-consumer-deploy`).

**Bentuk argumen.** Kedua binary hanya mengenali bentuk `--flag=value`. Bentuk terpisah spasi (`--project my-app`) diabaikan diam-diam tanpa pesan error, sehingga flag yang ditulis begitu dianggap tidak diisi. Pengecualiannya hanya flag tanpa nilai: `--force` / `-f`, `--help` / `-h`, dan `--version` / `-v`. Notasi `--project <NAME>` pada tabel di bawah adalah notasi tabel saja; seluruh contoh memakai bentuk `=` dan itu yang benar.

## `restforge-consumer`

Menjalankan Kafka consumer runtime untuk consumer yang sudah di-generate dalam sebuah project. Jika `--consumer` tidak diisi, seluruh consumer yang terdaftar di project akan dijalankan.

### Pattern

```
npx restforge-consumer --project=<NAME> --config=<FILE> [options]
```

### Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project target (alias: `--module <NAME>`) |
| `--config <FILE>` | Ya | - | File config `.env`. Nama file wajib berakhiran `.env`; tanpa flag ini CLI berhenti dengan exit code `1` |
| `--consumer <NAME>` | Tidak | semua | Consumer spesifik yang akan dijalankan |
| `--port <N>` | Tidak | `3001` | Port service untuk control API |
| `--license <KEY>` | Tidak | - | License key inline |
| `--license-server <URL>` | Tidak | auto | Endpoint custom license server |

### Contoh

```bat
:: Run seluruh consumer dalam project
npx restforge-consumer --project=my-app --config=config/db.env

:: Run consumer spesifik dengan port custom
npx restforge-consumer --project=my-app --config=config/db.env --consumer=order-events --port=3050
```

## `restforge-consumer-deploy`

Menghasilkan file PM2 ecosystem config untuk deploy multi-process consumer ke production environment.

### Pattern

```
npx restforge-consumer-deploy --project=<NAME> --config=<FILE> [options]
```

### Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project target (alias: `--module <NAME>`) |
| `--config <FILE>` | - | File config `.env` |

### Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--consumer <NAME>` | semua | Consumer spesifik (default: semua consumer di project) |
| `--port <N>` | `3001` | Port control API |
| `--output <DIR>` | `./deploy/` | Output directory untuk file PM2 ecosystem |
| `--license <KEY>` | - | License key inline |
| `--force` / `-f` | `false` | Timpa file yang sudah ada |

### Contoh

```bat
:: Generate ecosystem config untuk seluruh consumer
npx restforge-consumer-deploy --project=my-app --config=config/db.env --output=./deploy

:: Generate untuk satu consumer spesifik
npx restforge-consumer-deploy --project=my-app --consumer=order-events --config=config/db.env --force
```

---

**Lihat juga**: [`commands/`](./) · [`kafka/consumer-create`](./kafka/consumer-create.md) · [`README`](../README.md)
