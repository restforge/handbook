# `auth-service provision`

> Mendaftarkan aplikasi, permission, role OWNER, dan user owner ke layanan auth dari manifest permission, lalu mengirim email notifikasi ke owner baru.

## Pattern

```
npx restforge auth-service provision --manifest=<FILE> --owner-email=<EMAIL> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--manifest <FILE>` | Ya | - | File manifest permission, hasil [`manifest`](./manifest.md) |
| `--auth-config <FILE>` | Tidak | `auth.env` | File env layanan auth, sumber koneksi database |
| `--owner-email <EMAIL>` | Ya | - | Email user owner. Owner menerima email notifikasi di alamat ini dan memakainya untuk forgot password |
| `--owner-username <NAME>` | Tidak | `<app code huruf kecil>-owner` | Username user owner |
| `--owner-password <PASSWORD>` | Tidak | acak | Password user owner. Bila tidak diisi, password acak dibuat, ditampilkan, dan disimpan ke file kredensial |
| `--dry-run` | Tidak | `false` | Menampilkan rencana tanpa menulis ke database |

## Prasyarat

- [`auth-service bootstrap`](./bootstrap.md) sudah dijalankan, sehingga tabel layanan auth ada.
- Manifest sudah dibuat dengan [`manifest`](./manifest.md).
- Database pada file env layanan auth berupa PostgreSQL.
- SMTP di file env layanan auth sudah diisi bila email notifikasi owner ingin terkirim. Tanpa SMTP, `provision` tetap berjalan dan hanya menampilkan peringatan.

## Apa yang Dikerjakan

1. Mendaftarkan aplikasi dengan app code dari manifest.
2. Mendaftarkan satu permission untuk setiap pasangan resource dan aksi, lengkap dengan kategori, urutan, dan description aksi dari manifest.
3. Menyimpan description setiap resource, terpisah dari permission.
4. Membuat role `OWNER` dan memberinya seluruh permission aplikasi.
5. Membuat user owner dan memberinya role `OWNER`.
6. Mengirim email notifikasi ke owner yang baru dibuat.

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

## Username dan Email Owner

Username owner memakai app code dalam huruf kecil. Untuk app code `MYAPP`, username default adalah `myapp-owner`, dan nilai ini dapat diganti lewat `--owner-username`.

Email owner tidak punya nilai default dan wajib diisi lewat `--owner-email`, juga saat `--dry-run`. Pakai alamat yang benar-benar bisa menerima email, karena owner memakai alamat ini untuk menerima notifikasi dan link reset password. Perintah berhenti dengan error sebelum database disentuh bila flag ini kosong atau formatnya bukan alamat email.

Bila user owner sudah terdaftar, email-nya tidak diubah meskipun `--owner-email` berisi alamat lain. Output menampilkan catatan perbedaannya, misalnya `note: the existing owner email is owner@myapp.com; --owner-email new@myapp.com was not applied`.

## Email Notifikasi Owner

Setelah user owner baru dibuat, `provision` mengirim email ke alamat `--owner-email`. Email ini berisi nama aplikasi, app code, username owner, link ke halaman login, dan keterangan bahwa password yang terlupa dapat dipulihkan lewat Forgot password?. Password tidak ikut dikirim.

Email dikirim lewat SMTP di file env layanan auth, yaitu key `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, dan `SMTP_FROM`. Link login dibentuk dari `APP_BASE_URL`, misalnya `http://localhost:8000/login.html`. Kedua pengaturan ini dijelaskan di [Pengiriman Email](./init.md#pengiriman-email).

Kegagalan kirim tidak membatalkan provisioning, karena data sudah tersimpan di database. Bila SMTP belum diisi, masih berisi nilai contoh, atau server email menolak koneksi, output menampilkan peringatan beserta alasannya. Email tidak dikirim saat `--dry-run`, dan tidak dikirim ulang untuk owner yang sudah ada.

## Password Owner

Password owner yang sudah ada tidak pernah ditimpa. Bila user owner sudah terdaftar, `provision` melewatinya dan `--owner-password` diabaikan. Output menampilkan `owner user already exists, password left untouched`.

Bila user owner baru dibuat tanpa `--owner-password`, password acak muncul di output. Password itu juga ditulis sebagai teks polos ke `data-storage/credential/<app code huruf kecil>-owner.txt`, misalnya `myapp-owner.txt`, bersama app code, username, dan email owner. File yang sudah ada ditimpa.

Pindahkan password ke tempat yang aman, lalu hapus file tersebut. Folder `data-storage/credential/` berisi file `.gitignore` sehingga file kredensial tidak ikut ter-commit. File tidak dibuat bila password diberikan lewat `--owner-password`, bila user owner sudah ada, atau saat `--dry-run`.

## Dry Run

`--dry-run` membaca database secara read-only untuk membandingkan manifest dengan data yang sudah ada, lalu mencetak rencana: aplikasi, permission, description resource, role, dan owner beserta statusnya. Tidak ada data yang ditulis dan tidak ada email yang dikirim.

## Contoh

```bat
cd /d D:\projects\myapp
npx restforge auth-service provision --manifest=config/auth-manifest.json --owner-email=owner@myapp.com
npx restforge auth-service provision --manifest=config/auth-manifest.json --owner-email=owner@myapp.com --dry-run
npx restforge auth-service provision --manifest=config/auth-manifest.json ^
  --owner-email=owner@myapp.com --owner-username=admin-myapp
```

## Output

```
=== Provisioning complete: MYAPP ===
App:         created
Permissions: 4 created, 0 updated, 0 unchanged
Resources:   1 created, 0 updated, 0 unchanged
Role OWNER:  created - 4 new assignment(s)
Owner user:  created (myapp-owner <owner@myapp.com>)
Owner role:  assigned OWNER
Owner email: notification sent to owner@myapp.com

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

Bila SMTP belum diisi, baris `Owner email:` diganti peringatan berikut, lalu provisioning tetap selesai.

```
WARNING: owner notification email was not sent to owner@myapp.com: SMTP is not configured (SMTP_HOST, SMTP_USER, SMTP_PASSWORD in auth.env still hold the example values)
         Provisioning is complete. Configure SMTP in auth.env so the owner can use "Forgot password?".
```

## Error Umum

| Pesan | Penyebab |
|-------|----------|
| `Missing required flag: --owner-email` | Flag `--owner-email` tidak diisi. Flag ini wajib, juga untuk `--dry-run` |
| `--owner-email is required` atau `is not a valid email address` | Nilai `--owner-email` kosong atau bukan alamat email |
| `auth-service config file not found` | `config/auth.env` belum ada. Jalankan `auth-service init` |
| `auth-service currently supports PostgreSQL only` | Database pada file env bukan PostgreSQL |
| `Cannot read manifest file` | File manifest tidak ditemukan. Periksa path `--manifest` |
| `is not valid JSON` atau `Invalid manifest` | Isi manifest tidak valid. Perbaiki file atau buat ulang dengan `manifest` |

---

**Lihat juga**: [`auth-service/`](./README.md) · [`manifest`](./manifest.md) · [`bootstrap`](./bootstrap.md) · [`commands/`](../../README.md) · [`README`](../../../README.md)
