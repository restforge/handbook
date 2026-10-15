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
| `--database <TYPE>` | auto-detect | Tipe database (`postgres`, `mysql`, `oracle`). SQLite ditolak khusus untuk dashboard. |
| `--config <ENV>` | - | File config database (.env) yang `DB_TYPE`-nya menentukan dialect bila `--database` tidak diisi |
| `--force` | `false` | Timpa file yang sudah ada. Versi lama diarsipkan ke `.restforge/archive/` |
| `--skip-sql-validation` | `false` | Lewati validasi keyword SQL |
| `--validate-only` | `false` | Jalankan pipeline validasi penuh lalu berhenti sebelum menulis apa pun |

## Resolusi Tipe Database

Dialect ditentukan dengan urutan berikut, sama seperti [`endpoint create`](../endpoint/create.md).

1. Nilai `--database` dipakai bila diisi.
2. Bila `--database` kosong, CLI membaca `DB_TYPE` dari file `--config`, atau dari default config yang disetel lewat `config set-default`.
3. Bila tidak ada config yang bisa dibaca, dialect jatuh ke `postgres`.

Dialect hasil deteksi ditampilkan di baris `Database` pada output dengan keterangan `(auto-detected from --config)` atau `(auto-detected from default config)`. Nilai `--database` di luar daftar dialect ditolak dengan pesan yang menyebut nilai yang sah, dan file `--config` yang tidak ditemukan juga ditolak.

SQLite ditolak di awal command, baik dari `--database` maupun dari hasil deteksi, sehingga tidak ada file yang ditulis.

## Mode Validasi Saja

Dengan `--validate-only`, CLI menjalankan pipeline validasi yang sama persis dengan jalur generate lalu berhenti tepat sebelum penulisan pertama. Tidak ada file module, metadata, maupun entri registry project yang ditulis atau diubah. Exit code `0` berarti payload valid, exit code selain `0` berarti validator menolak payload dengan pesan yang identik dengan jalur generate.

Karena tidak ada yang ditulis, pemeriksaan konflik registry dilewati dan `--force` tidak diperlukan sekalipun dashboard dengan nama yang sama sudah terdaftar.

**Ketersediaan versi**: flag ini masuk ke source platform setelah rilis 5.5.5. Pada versi yang lebih lama CLI menjawab `Unknown flag: --validate-only`.

## Contoh

```bat
npx restforge dashboard create --project=mini-inventory --name=dash-sales --payload=dashboard-sales.json
```

Contoh berikut memakai dialect dari `DB_TYPE` di file config tertentu.

```bat
npx restforge dashboard create --project=mini-inventory --name=dash-sales --payload=dashboard-sales.json --config=mysql.env
```

Validasi payload tanpa menghasilkan file apa pun:

```bat
npx restforge dashboard create --project=mini-inventory --name=dash-sales --payload=dashboard-sales.json --validate-only
```

---

**Lihat juga**: [`dashboard/`](./) · [`commands/`](../) · [`README`](../../../README.md)
