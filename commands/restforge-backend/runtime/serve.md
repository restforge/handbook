# `serve`

> Menjalankan runtime HTTP server untuk project RESTForge yang sudah di-generate.

## Pattern

```
npx restforge serve --project=<NAME> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--project <NAME>` | - | Nama project yang akan dijalankan |

## Flag Opsional

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

## Contoh

```bat
:: Start basic
npx restforge serve --project=my-project

:: Start dengan port custom dan cluster
npx restforge serve --project=my-project --port=8080 --cluster --workers=4

:: Start dengan license inline dan custom config
npx restforge serve --project=my-project --config=production.env --license=AAAA-BBBB-CCCC-DDDD
```

## Exit Code

Server yang gagal start berhenti dengan exit code yang membedakan error konfigurasi dari kegagalan yang bisa pulih dengan restart.

| Exit code | Kondisi |
|-----------|---------|
| `78` | Error konfigurasi saat startup, misalnya file `--config` tidak ditemukan, `SERVER_ADDRESS` atau `SERVER_PORT` kosong, package seperti `bullmq` atau `oracledb` belum terpasang padahal dibutuhkan env, license tidak valid, atau project tidak bisa dimuat. Restart tidak membantu sampai konfigurasinya diperbaiki |
| `1` | Crash atau kegagalan yang bisa pulih dengan restart, misalnya Redis atau license server belum bisa dihubungi, port masih dipakai proses lain, dan error tak terduga saat server berjalan |

Pada cluster mode, worker yang berhenti karena error konfigurasi tidak dijalankan ulang. Proses utama ikut berhenti dengan exit code `78`.

## Menjalankan dengan PM2

PM2 menjalankan ulang proses setiap kali proses berhenti dengan exit code bukan nol. Tanpa pengaturan tambahan, server yang konfigurasinya salah dijalankan ulang terus-menerus, dan pesan penyebabnya tertimbun di file log PM2. Opsi `stop_exit_codes` membuat PM2 berhenti menjalankan ulang proses untuk exit code tertentu.

```js
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'my-project',
      script: 'node_modules/@restforgejs/platform/server.js',
      args: 'serve --project=my-project --config=production.env',
      autorestart: true,
      stop_exit_codes: [78]
    }
  ]
};
```

Dengan pengaturan ini, server yang gagal karena error konfigurasi berstatus `stopped` di `pm2 status`, dan pesan error-nya menjadi baris terakhir di `pm2 logs my-project`. Setelah konfigurasi diperbaiki, jalankan lagi dengan `pm2 restart my-project`. Crash dengan exit code `1` tetap dijalankan ulang oleh PM2.

File `ecosystem.config.js` yang dibuat [`fast-track`](../fast-track.md) dan `restforge-consumer-deploy` sudah memuat `stop_exit_codes: [78]`. Opsi ini perlu ditambahkan sendiri pada `ecosystem.config.js` yang ditulis manual.

---

**Lihat juga**: [`runtime/`](./) · [`commands/`](../) · [`README`](../../../README.md)
