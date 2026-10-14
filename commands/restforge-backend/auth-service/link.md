# `auth-service link`

> Menghubungkan project aplikasi ke layanan auth: menulis app code dan kunci verifikasi token ke env aplikasi, lalu memasang blok `authGuard` ke RDF project.

## Pattern

```
npx restforge auth-service link --project=<NAME> --app-code=<CODE> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project aplikasi |
| `--app-code <CODE>` | Ya | - | App code di layanan auth. Huruf besar, angka, dan garis bawah, diawali huruf |
| `--config <FILE>` | Tidak | `db-connection.env` | File env aplikasi |
| `--auth-config <FILE>` | Tidak | `auth.env` | File env layanan auth, sumber `JWT_SECRET` atau public key |

`--config` harus berbeda dari `--auth-config`.

## Prasyarat

- [`auth-service init`](./init.md) sudah dijalankan.
- Project belum memasang [`project auth`](../project/auth.md).

## Apa yang Dikerjakan

1. Membaca kunci verifikasi token dari env layanan auth: `JWT_SECRET` untuk `HS256`, atau `JWT_PUBLIC_KEY_PATH` untuk `RS256`.
2. Menulis `AUTH_APP_CODE` dan kunci tersebut ke env aplikasi. Hanya nilai yang berbeda yang diubah, baris lain tetap.
3. Memasang blok [`authGuard`](../../../catalogs/rdf/auth-guard.md) dengan `appCode` ke setiap RDF yang menjadi sumber endpoint project. Blok baru mendapat `enabled: true`. Pada blok yang sudah ada, `enabled: false` dan `publicPaths` dipertahankan, dan bagian lain RDF tidak berubah.

RDF layanan auth sendiri tidak disentuh. Perintah ini idempoten: menjalankannya lagi dengan nilai yang sama tidak mengubah file apa pun.

`link` sebaiknya dijalankan sebelum endpoint pertama dibuat. Project tanpa endpoint tetap ditautkan: env aplikasi ditulis, langkah 3 dilewati, dan perintah selesai tanpa error. Sejak `AUTH_APP_CODE` ada di env aplikasi, [`payload generate`](../payload/generate.md) memasang `authGuard` ke RDF yang belum punya blok ini, dan [`endpoint create`](../endpoint/create.md) menambahkan resource-nya ke `config/auth-manifest.json`. Tabel baru tidak memerlukan `link` lagi.

Pada project yang sudah punya endpoint, `authGuard` baru berlaku setelah endpoint dibuat ulang. Output menampilkan perintah `endpoint create` yang perlu dijalankan untuk setiap RDF yang berubah.

## Contoh

```bat
cd /d D:\projects\myapp
npx restforge auth-service link --project=myapp --app-code=MYAPP
npx restforge auth-service link --project=myapp --app-code=MYAPP ^
  --config=db-connection.env --auth-config=auth.env
```

## Output

Project yang belum punya endpoint menghasilkan output berikut.

```
Application environment (config/db-connection.env):
  updated: AUTH_APP_CODE, JWT_SECRET (JWT_ALGORITHM=HS256)

Project 'myapp' has no endpoint yet. RDF files created from now on get authGuard (appCode MYAPP) automatically:
  "payload generate" adds it to new RDF files, and "endpoint create" adds the resource to config/auth-manifest.json.

Next:
  npx restforge payload generate --table=<table>
  npx restforge endpoint create --project=myapp --name=<name> --payload=<table>.json
```

Project yang sudah punya endpoint menghasilkan output berikut.

```
Application environment (config/db-connection.env):
  updated: AUTH_APP_CODE, JWT_SECRET (JWT_ALGORITHM=HS256)

authGuard (appCode MYAPP) in RDF files of project 'myapp':
  updated (1):
    - payload/item.json
  An authGuard with "enabled": false is kept disabled. RDF files created from now on get authGuard automatically.

authGuard takes effect after the endpoints are regenerated. Run "npx restforge endpoint create" again for:
  npx restforge endpoint create --project=myapp --name=item --payload=item.json
"endpoint create" also adds the resources to config/auth-manifest.json.
```

## Error Umum

| Pesan | Penyebab |
|-------|----------|
| `Application config file not found` | File env aplikasi tidak ada. Berikan lewat `--config` |
| `auth-service config file not found` | `config/auth.env` belum ada. Jalankan `auth-service init` |
| `--project=auth-service is the auth service itself` | Project yang diberikan adalah layanan auth, bukan aplikasi |
| `already uses the embedded login from "project auth"` | Project memakai `project auth`. Kedua model auth memakai kunci JWT yang sama sehingga tidak dapat digabung |
| Error pada `--app-code` | Nilai memuat huruf kecil, tanda hubung, atau diawali angka |

Semua error muncul sebelum file apa pun diubah.

---

**Lihat juga**: [`auth-service/`](./README.md) · [`manifest`](./manifest.md) · [`payload generate`](../payload/generate.md) · [`catalogs/rdf/auth-guard`](../../../catalogs/rdf/auth-guard.md) · [`commands/`](../../README.md) · [`README`](../../../README.md)
