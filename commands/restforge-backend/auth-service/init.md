# `auth-service init`

> Memasang file layanan auth ke folder project aplikasi dan membuat file env layanan. Perintah ini tidak menyentuh database.

## Pattern

```
npx restforge auth-service init [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--app-config <FILE>` | Tidak | `db-connection.env` | File env aplikasi, sumber nilai `DB_*` dan `LICENSE` |
| `--config <FILE>` | Tidak | `auth.env` | File env layanan auth yang dibuat di folder `config/` |
| `--port <N>` | Tidak | `3100` | `SERVER_PORT` layanan auth |
| `--jwt-algorithm <ALG>` | Tidak | `HS256` | `HS256` atau `RS256`. `RS256` membuat keypair di `config/keys/` |
| `--force` | Tidak | `false` | Timpa file layanan auth yang sudah ada. File lama diarsipkan lebih dulu |

`--config` harus berbeda dari `--app-config`, karena layanan auth memakai file env sendiri.

## Prasyarat

- File env aplikasi ada di folder kerja atau di `config/`.
- Database pada file env aplikasi berupa PostgreSQL.

## Apa yang Dikerjakan

1. Memeriksa dialect database dari `--app-config`. Dialect selain PostgreSQL ditolak sebelum file apa pun ditulis.
2. Menyalin definisi schema, definisi endpoint, query, data awal, komponen, plugin, view, dan handler processor layanan auth ke folder project. File yang sudah ada dilewati.
3. Membuat `config/<--config>` berisi `DB_*` dan `LICENSE` dari env aplikasi, `SERVER_PORT`, dan kunci JWT.
4. Mencatat `nodemailer` dan `ejs` sebagai dependency di `package.json` project.

Kunci JWT bergantung pada `--jwt-algorithm`. Mode `HS256` membuat `JWT_SECRET` acak. Mode `RS256` membuat keypair RSA di `config/keys/` dan mengisi path private dan public key di file env.

Daftar file yang disalin ada di [Struktur Folder](./README.md#struktur-folder).

## Menjalankan Ulang dengan `--force`

Tanpa `--force`, `init` kedua melewati semua file yang sudah ada. Dengan `--force`, file yang sudah ada diarsipkan ke `.restforge/archive/` lalu ditimpa, sehingga perubahan lokal pada file layanan auth tetap bisa dipulihkan dari arsip.

Kunci JWT yang sudah ada dipertahankan. `JWT_SECRET` (atau keypair RS256) tidak dibuat ulang, sehingga token yang sudah terbit dan aplikasi yang sudah di-`link` tetap valid. Output menampilkan baris `Existing JWT key preserved`.

## Contoh

```bat
cd /d D:\projects\myapp
npx restforge auth-service init
npx restforge auth-service init --app-config=db-connection.env --port=3110
npx restforge auth-service init --jwt-algorithm=RS256
npx restforge auth-service init --force
```

Setelah `init`, jalankan `npm install` agar `nodemailer` dan `ejs` terpasang.

## Output

```
auth-service files:
  written: 75

auth-service environment (config/auth.env):
  written: DB_* and LICENSE copied from config/db-connection.env, SERVER_PORT=3100, JWT_ALGORITHM=HS256
  JWT_SECRET: generated (64 random bytes)

Runtime dependencies (nodemailer, ejs):
  package.json updated, dependencies added: nodemailer, ejs

auth-service initialized. The database was not touched.
Next steps:
  1. npm install
  2. npx restforge auth-service bootstrap
```

## Error Umum

| Pesan | Penyebab |
|-------|----------|
| `Application config file not found` | File env aplikasi tidak ada. Berikan lewat `--app-config` |
| `auth-service currently supports PostgreSQL only` | Database aplikasi bukan PostgreSQL |
| `--config must differ from --app-config` | Kedua flag menunjuk file yang sama |

Ketiga error muncul sebelum file apa pun ditulis.

---

**Lihat juga**: [`auth-service/`](./README.md) · [`bootstrap`](./bootstrap.md) · [`commands/`](../../README.md) · [`README`](../../../README.md)
