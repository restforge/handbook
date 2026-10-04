# `auth-service provision`

> Mendaftarkan aplikasi, permission, role OWNER, dan user owner ke layanan auth dari manifest permission.

## Pattern

```
npx restforge auth-service provision --manifest=<FILE> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--manifest <FILE>` | Ya | - | File manifest permission, hasil [`manifest`](./manifest.md) |
| `--auth-config <FILE>` | Tidak | `auth.env` | File env layanan auth, sumber koneksi database |
| `--owner-email <EMAIL>` | Tidak | `<owner-username>@<app code huruf kecil>.local` | Email user owner |
| `--owner-username <NAME>` | Tidak | `<app code huruf kecil>-owner` | Username user owner |
| `--owner-password <PASSWORD>` | Tidak | acak | Password user owner. Bila tidak diisi, password acak dibuat, ditampilkan, dan disimpan ke file kredensial |
| `--dry-run` | Tidak | `false` | Menampilkan rencana tanpa menulis ke database |

## Prasyarat

- [`auth-service bootstrap`](./bootstrap.md) sudah dijalankan, sehingga tabel layanan auth ada.
- Manifest sudah dibuat dengan [`manifest`](./manifest.md).
- Database pada file env layanan auth berupa PostgreSQL.

## Apa yang Dikerjakan

1. Mendaftarkan aplikasi dengan app code dari manifest.
2. Mendaftarkan satu permission untuk setiap pasangan resource dan aksi, lengkap dengan kategori, urutan, dan description aksi dari manifest.
3. Menyimpan description setiap resource, terpisah dari permission.
4. Membuat role `OWNER` dan memberinya seluruh permission aplikasi.
5. Membuat user owner dan memberinya role `OWNER`.

Seluruh langkah idempoten. Menjalankan `provision` lagi hanya menambah yang belum ada, dan permission yang berubah di manifest diperbarui.

Bila manifest memakai aksi yang belum terdaftar sebagai action type di database layanan auth, misalnya `PRINT` atau `CHANGE_STATUS` pada database yang di-bootstrap sebelum kedua aksi itu tersedia, `provision` menambahkannya lebih dulu. Output menampilkan baris `Action types: added PRINT, CHANGE_STATUS`. Action type menentukan urutan dan label kolom di halaman Assign Permissions.

## Description Resource dan Aksi

Description resource dan description aksi diambil dari manifest. Description resource tampil di baris resource pada halaman Assign Permissions, sedangkan description aksi tampil sebagai tooltip checkbox aksi itu. Bila description di manifest diubah atau dihapus, `provision` berikutnya memperbarui data yang tersimpan. Output menampilkan jumlahnya pada baris `Resources:`.

Database yang di-bootstrap sebelum description resource tersedia disiapkan pada `provision` pertama, dengan output `Resources:   created table auth.app_resource`. Permission lama yang berisi salinan description resource ikut dikosongkan, kecuali permission yang punya description aksi di manifest.

Halaman Assign Permissions membaca description resource lewat endpoint `permission` layanan auth. Layanan auth yang dipasang sebelum fitur ini perlu definisi endpoint `permission` versi baru. Jalankan [`init --force`](./init.md#menjalankan-ulang-dengan---force) dengan flag yang sama seperti pemasangan awal, misalnya `--port`, lalu buat ulang endpoint tersebut:

```bat
npx restforge endpoint create --project=auth-service --name=permission --payload=auth_permission.json --config=auth.env --database=postgres --force
```

Selama endpoint belum dibuat ulang, matriks tetap berjalan dan area description resource dibiarkan kosong.

## Nilai Default Owner

Username owner memakai app code dalam huruf kecil. Untuk app code `MYAPP`, username default adalah `myapp-owner` dan email default adalah `myapp-owner@myapp.local`. Kedua nilai dapat diganti lewat `--owner-username` dan `--owner-email`.

## Password Owner

Password owner yang sudah ada tidak pernah ditimpa. Bila user owner sudah terdaftar, `provision` melewatinya dan `--owner-password` diabaikan. Output menampilkan `owner user already exists, password left untouched`.

Bila user owner baru dibuat tanpa `--owner-password`, password acak muncul di output. Password itu juga ditulis sebagai teks polos ke `data-storage/credential/<app code huruf kecil>-owner.txt`, misalnya `myapp-owner.txt`, bersama app code, username, dan email owner. File yang sudah ada ditimpa.

Pindahkan password ke tempat yang aman, lalu hapus file tersebut. Folder `data-storage/credential/` berisi file `.gitignore` sehingga file kredensial tidak ikut ter-commit. File tidak dibuat bila password diberikan lewat `--owner-password`, bila user owner sudah ada, atau saat `--dry-run`.

## Dry Run

`--dry-run` membaca database secara read-only untuk membandingkan manifest dengan data yang sudah ada, lalu mencetak rencana: aplikasi, permission, description resource, role, dan owner beserta statusnya. Tidak ada data yang ditulis.

## Contoh

```bat
cd /d D:\projects\myapp
npx restforge auth-service provision --manifest=config/auth-manifest.json
npx restforge auth-service provision --manifest=config/auth-manifest.json --dry-run
npx restforge auth-service provision --manifest=config/auth-manifest.json ^
  --owner-email=owner@myapp.com
```

## Output

```
=== Provisioning complete: MYAPP ===
App:         created
Permissions: 4 created, 0 updated, 0 unchanged
Resources:   1 created, 0 updated, 0 unchanged
Role OWNER:  created - 4 new assignment(s)
Owner user:  created (myapp-owner <myapp-owner@myapp.local>)
Owner role:  assigned OWNER

============================================================
  GENERATED OWNER PASSWORD:
    username: myapp-owner
    password: <password acak>
  Saved to: data-storage/credential/myapp-owner.txt
  WARNING: this file contains the password in plain text.
  Store the password in a secure place, then delete the file.
============================================================

Login check: POST /api/auth-service/session/login
  { "app_code": "MYAPP", "username": "myapp-owner", "password": "<owner password>" }
```

## Error Umum

| Pesan | Penyebab |
|-------|----------|
| `auth-service config file not found` | `config/auth.env` belum ada. Jalankan `auth-service init` |
| `auth-service currently supports PostgreSQL only` | Database pada file env bukan PostgreSQL |
| `Cannot read manifest file` | File manifest tidak ditemukan. Periksa path `--manifest` |
| `is not valid JSON` atau `Invalid manifest` | Isi manifest tidak valid. Perbaiki file atau buat ulang dengan `manifest` |

---

**Lihat juga**: [`auth-service/`](./README.md) · [`manifest`](./manifest.md) · [`bootstrap`](./bootstrap.md) · [`commands/`](../../README.md) · [`README`](../../../README.md)
