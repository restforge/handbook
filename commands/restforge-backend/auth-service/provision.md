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
| `--owner-password <PASSWORD>` | Tidak | acak | Password user owner. Bila tidak diisi, password acak dibuat dan ditampilkan satu kali |
| `--dry-run` | Tidak | `false` | Menampilkan rencana tanpa menulis ke database |

## Prasyarat

- [`auth-service bootstrap`](./bootstrap.md) sudah dijalankan, sehingga tabel layanan auth ada.
- Manifest sudah dibuat dengan [`manifest`](./manifest.md).
- Database pada file env layanan auth berupa PostgreSQL.

## Apa yang Dikerjakan

1. Mendaftarkan aplikasi dengan app code dari manifest.
2. Mendaftarkan satu permission untuk setiap pasangan resource dan aksi, lengkap dengan kategori dan urutan dari manifest.
3. Membuat role `OWNER` dan memberinya seluruh permission aplikasi.
4. Membuat user owner dan memberinya role `OWNER`.

Seluruh langkah idempoten. Menjalankan `provision` lagi hanya menambah yang belum ada, dan permission yang berubah di manifest diperbarui.

## Nilai Default Owner

Username owner memakai app code dalam huruf kecil. Untuk app code `MYAPP`, username default adalah `myapp-owner` dan email default adalah `myapp-owner@myapp.local`. Kedua nilai dapat diganti lewat `--owner-username` dan `--owner-email`.

## Password Owner

Password owner yang sudah ada tidak pernah ditimpa. Bila user owner sudah terdaftar, `provision` melewatinya dan `--owner-password` diabaikan. Output menampilkan `owner user already exists, password left untouched`.

Bila user owner baru dibuat tanpa `--owner-password`, password acak muncul satu kali di output dan tidak disimpan dalam bentuk asli. Simpan password itu saat muncul.

## Dry Run

`--dry-run` membaca database secara read-only untuk membandingkan manifest dengan data yang sudah ada, lalu mencetak rencana: aplikasi, permission, role, dan owner beserta statusnya. Tidak ada data yang ditulis.

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
Role OWNER:  created - 4 new assignment(s)
Owner user:  created (myapp-owner <myapp-owner@myapp.local>)
Owner role:  assigned OWNER

============================================================
  GENERATED OWNER PASSWORD (shown once - store it now):
    username: myapp-owner
    password: <password acak>
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
