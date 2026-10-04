# `auth-service bootstrap`

> Membuat schema, tabel, dan data awal layanan auth di database aplikasi, lalu membuat modul layanan auth.

## Pattern

```
npx restforge auth-service bootstrap [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Tidak | `auth.env` | File env layanan auth yang dibuat oleh `init` |
| `--reset` | Tidak | `false` | Membuang dan membuat ulang tabel layanan auth meskipun sudah berisi data (destruktif) |

## Prasyarat

- [`auth-service init`](./init.md) sudah dijalankan, sehingga file layanan auth dan `config/auth.env` ada.
- Database pada file env berupa PostgreSQL. Server database dapat dijangkau dengan kredensial di file env.

## Apa yang Dikerjakan

1. Membuat database bila belum ada, lalu membuat schema `auth`.
2. Memeriksa guard data. Bila tabel layanan auth sudah berisi data dan `--reset` tidak diberikan, perintah berhenti.
3. Membuat tabel layanan auth dari definisi schema di `schema/auth/`.
4. Memasukkan data awal: aplikasi SYSTEM, action type, permission, role, dan user `superadmin`.
5. Membuat endpoint dan processor layanan auth dengan nama project `auth-service`.
6. Mengganti password `superadmin` dan `app_secret` aplikasi SYSTEM dengan nilai acak, menampilkannya di output, lalu menyimpannya ke file kredensial.

Langkah 3 sampai 5 berhenti pada kegagalan pertama dan menampilkan langkah yang gagal.

## Guard Data

`bootstrap` melindungi database yang sudah berisi data layanan auth, misalnya hasil restore. Jalankan kedua kalinya tanpa `--reset` dan perintah berhenti dengan exit code bukan nol, dengan pesan yang menyebut `--reset`. Isi tabel dan file hasil generate tidak berubah.

## Reset

`--reset` membuang dan membuat ulang tabel di schema `auth`, lalu mengisi ulang data awal dan membuat password baru. User, role, permission, dan aplikasi yang sudah terdaftar di layanan auth hilang, sehingga `provision` perlu dijalankan lagi.

`--reset` hanya menyentuh schema `auth`. Tabel dan data aplikasi di schema lain, termasuk `public`, tidak berubah.

## File Kredensial

Password `superadmin` dan `app_secret` aplikasi SYSTEM muncul di akhir output. Keduanya juga ditulis sebagai teks polos ke `data-storage/credential/auth-service-superadmin.txt` di folder project. File itu ditimpa setiap kali `bootstrap` membuat kredensial baru.

Pindahkan kredensial ke tempat yang aman, lalu hapus file tersebut. Folder `data-storage/credential/` berisi file `.gitignore` sehingga file kredensial tidak ikut ter-commit. Bila kredensial hilang, jalankan `bootstrap --reset`.

## Contoh

```bat
cd /d D:\projects\myapp
npx restforge auth-service bootstrap
npx restforge auth-service bootstrap --config=auth.env
npx restforge auth-service bootstrap --reset
```

## Output

```
================================================================
  auth-service bootstrap complete

  superadmin username : superadmin
  superadmin password : <password acak>
  SYSTEM app_secret   : <secret acak>

  Saved to: data-storage/credential/auth-service-superadmin.txt
  WARNING: this file contains the credentials in plain text.
  Store the credentials in a secure place, then delete the file.
================================================================

Next step:
  npx restforge serve --project=auth-service --config=auth.env
```

## Error Umum

| Pesan | Penyebab |
|-------|----------|
| `auth-service config file not found` | `config/auth.env` belum ada. Jalankan `auth-service init` |
| `auth-service files not found` | File hasil `init` tidak lengkap. Jalankan `auth-service init` |
| `auth-service currently supports PostgreSQL only` | Database pada file env bukan PostgreSQL |
| `Cannot connect to postgres` | Server database tidak dapat dijangkau atau kredensial salah |
| Berhenti di langkah 2 dengan pesan yang menyebut `--reset` | Tabel layanan auth sudah berisi data |

---

**Lihat juga**: [`auth-service/`](./README.md) · [`init`](./init.md) · [`provision`](./provision.md) · [`commands/`](../../README.md) · [`README`](../../../README.md)
