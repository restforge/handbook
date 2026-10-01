# `project auth`

> Memasang auth extension (SDF, tabel DB, middleware, router, 7 processor, env auth, dependency) ke project RESTForge yang sudah ada.

## Pattern

```
npx restforge project auth --create --project=<NAME> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--create` | Ya | `false` | Trigger instalasi (wajib disertakan) |
| `--project <NAME>` | Ya* | — | Nama project target |
| `--name <NAME>` | Ya* | — | Alias dari `--project` |
| `--schema-path <DIR>` | Tidak | `./schema` | Folder output file SDF auth |
| `--config <FILE>` | Tidak | `config/db-connection.env` | File konfigurasi DB untuk langkah migrate |
| `--force` | Tidak | `false` | Timpa file yang sudah ada (backup tetap dibuat) |

*`--project` atau `--name` wajib diisi, pilih salah satu.

## Prasyarat

- Project target sudah ada (jalankan `endpoint create` lebih dulu)
- DB aktif dan terkonfigurasi di `--config`

## Apa yang Dikerjakan

Command ini mengeksekusi langkah berikut secara berurutan:

1. Generate SDF auth (prefix `rfx`) ke `--schema-path`
2. Buat tabel auth di DB via dbschema-kit (idempoten, `IF NOT EXISTS`)
3. Tulis middleware + router auth
4. Tulis 7 processor: `register`, `login`, `google`, `refresh`, `logout`, `me`, `reset-password`
5. Inject variabel env auth (`JWT_SECRET` acak, `GOOGLE_CLIENT_ID` kosong) ke file config
6. Catat dependency `bcrypt` + `jsonwebtoken` ke `package.json` project

## Alur Tenant dan Kepemilikan

Auth ini berbasis tenant. Ketika seseorang mendaftar (lewat `register` maupun sign in
with Google untuk pertama kali), ia otomatis menjadi pemilik (owner) sebuah tenant baru.
Tenant adalah ruang milik organisasi atau tim pengguna tersebut.

Setelah login berhasil, data pengguna pada respons menyertakan identitas tenant dan peran
pengguna, sehingga aplikasi frontend dapat menyesuaikan akses dan tampilan.

Mengundang pengguna lain sebagai anggota tenant yang sama direncanakan pada versi
berikutnya. Saat ini setiap pendaftaran menghasilkan satu pemilik tenant.

> Project yang sudah memasang auth versi sebelumnya perlu langkah penyesuaian data saat
> upgrade. Bila memungkinkan, pasang pada database baru.

## Contoh

```bat
npx restforge project auth --create --project=myapp
npx restforge project auth --create --name=myapp --schema-path=./schema
npx restforge project auth --create --project=myapp --force
```

## Sign in with Google

Processor `google` dan route `POST /api/<project>/rfx_auth/google` ikut digenerate, namun
**tidak aktif sampai `GOOGLE_CLIENT_ID` diisi**. Selama kosong, endpoint mengembalikan
`501 Google login is not configured`.

### Kontrak endpoint

| Item | Nilai |
|------|-------|
| Method + path | `POST /api/<project>/rfx_auth/google` |
| Body | `{ "credential": "<google-id-token>" }` |
| Respons sukses | `{ success, data: { access_token, refresh_token, token_type, expires_in, user } }` |

`credential` adalah **Google ID token (JWT)** yang dihasilkan browser oleh Google Identity
Services (GIS), bukan input yang diketik user. Backend memverifikasi token ke endpoint
`tokeninfo` Google, memastikan `aud === GOOGLE_CLIENT_ID` dan `email_verified`, lalu
find-or-create user dengan identitas = email (disatukan dengan akun manual ber-email sama).
User baru via Google menjadi owner tenant baru (sama seperti `register`), sedangkan user
lama langsung login. Backend kemudian menerbitkan access + refresh token sendiri (sama
seperti `login`).

### Cara aktivasi

1. Buat **OAuth 2.0 Client ID** (tipe *Web application*) di Google Cloud Console.
2. Daftarkan origin frontend (mis. `http://localhost:3000`) di *Authorized JavaScript origins*.
3. Isi `GOOGLE_CLIENT_ID=<client-id>` di file config (key sudah ditulis kosong pada langkah 5).
4. Frontend: muat script GIS, tampilkan tombol, lalu kirim `response.credential` ke endpoint.

```html
<script src="https://accounts.google.com/gsi/client" async></script>
<div id="g_id_onload"
     data-client_id="GOOGLE_CLIENT_ID_ANDA"
     data-callback="onGoogleCredential"></div>
<div class="g_id_signin" data-type="standard"></div>

<script>
  // GIS memanggil callback ini dengan response.credential = Google ID token
  async function onGoogleCredential(response) {
    const res = await fetch('http://127.0.0.1:3000/api/myapp/rfx_auth/google', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ credential: response.credential })
    });
    const { data } = await res.json();
    // simpan data.access_token / data.refresh_token / data.user lalu redirect
  }
</script>
```

Konsumen yang memakai JavaScript SDK (`project sdk`) cukup memanggil
`client.auth.google(response.credential)` tanpa menulis `fetch` manual.

## Catatan

Undang anggota tenant, penegakan RBAC penuh, dan frontend Designer (tombol login
otomatis) belum dicakup versi ini.

---

**Lihat juga**: [`project/`](./) · [`commands/`](../) · [`README`](../../../README.md)
