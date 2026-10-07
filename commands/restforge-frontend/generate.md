# `npx restforge-designer generate`

> Generate aplikasi frontend (HTML/JS/CSS) dari UDF.

## Sintaks

```bash
npx restforge-designer generate --payload <PAYLOAD> --output <OUTPUT> [OPTIONS]
```

## Flag

| Flag | Tipe | Default | Wajib | Keterangan |
|------|------|---------|:-----:|-----------|
| `-p`, `--payload <PAYLOAD>` | path | — | ✓ | Path ke file UDF JSON |
| `-o`, `--output <OUTPUT>` | path | — | ✓ | Folder output untuk file yang di-generate |
| `--plugin <PLUGIN>` | string | dari `appConfig.plugin` | ✗ | Override plugin ID dari UDF |
| `--overwrite` | flag | `false` | ✗ | Timpa file existing di folder output. Versi lama diarsipkan ke `.restforge/archive/` |
| `--scope <SCOPE>` | enum | `app` | ✗ | Scope generate: `app` (semua) atau `form` (satu halaman saja) |
| `--page <PAGE>` | string | — | ✓ jika `--scope=form` | Page ID target untuk scope `form` |
| `--skip-shared` | flag | `false` | ✗ | Skip generate shared files (`index.html`, `js/common.js`, CSS, dll.) |
| `--plugins-dir <PATH>` | string | auto-detect | ✗ | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |

## Contoh Penggunaan

### Generate Lengkap (Scope `app`)

```bash
npx restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app
```

Generate semua page, shared files, dan plugin assets.

### Overwrite Folder Existing

```bash
npx restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --overwrite
```

Tanpa `--overwrite`, generate gagal jika folder output sudah berisi file.

### Generate Satu Page Saja (Scope `form`)

```bash
npx restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --scope=form --page=contact --overwrite
```

Hanya regenerate `contact.html` + `js/contact.js`. Shared files tetap di-generate kecuali ditambah `--skip-shared`.

### Skip Shared Files

```bash
npx restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --scope=form --page=contact --skip-shared --overwrite
```

Hanya page yang ditarget yang di-generate. `index.html`, `app-start.bat`, `js/common.js` tidak disentuh.

### Override Plugin dari UDF

```bash
npx restforge-designer generate --payload=payload/contact-mgmt.json --output=./apps/contact-mgmt-app --plugin=vanilla-js-auth --overwrite
```

Generate dengan plugin berbeda dari yang tertulis di `appConfig.plugin`. Berguna untuk membandingkan output antar plugin tanpa mengubah UDF.

## Scope Generate

| Scope | Behavior |
|-------|----------|
| `app` (default) | Generate semua page + shared files + plugin assets |
| `form` | Generate satu page saja (`--page=<id>` wajib). Shared files tetap (kecuali `--skip-shared`) |

Scope `form` berguna untuk:

- Iterasi cepat saat develop satu page tanpa regenerate semua
- Update parsial di production tanpa overwrite shared files yang sudah custom

## File yang Di-generate

### Shared Files (Per Aplikasi)

| File | Fungsi |
|------|--------|
| `index.html` | Homepage atau redirect ke `homepage` page |
| `js/common.js` | Utility bersama (notifikasi, format, validasi) |
| `css/global-list.css` | Design system CSS |
| `app-start.bat` | Script Windows untuk run `npx serve -l <port>` di port dari `appConfig.port` |

### Per-Page Files

| File | Fungsi |
|------|--------|
| `{pageId}.html` | Halaman CRUD atau dashboard |
| `js/{pageId}.js` | Logic CRUD: DataTables init, save/load/delete, validasi |

### Halaman Registrasi Plugin `vanilla-js-auth`

Plugin `vanilla-js-auth` menulis halaman registrasi publik bersama halaman login. Keempat file berikut dibuka tanpa login, sehingga tidak memuat `js/common.js`.

| File | Fungsi |
|------|--------|
| `register.html` | Form pendaftaran dengan Full Name, Email Address, dan WhatsApp Number |
| `js/register.js` | Logika form register dan pembacaan status registrasi |
| `set-password.html` | Halaman penetapan password dari link di email aktivasi |
| `js/set-password.js` | Logika halaman set password |

Link Create one di `login.html` hanya tampil saat registrasi terbuka. Halaman login membaca status registrasi dari layanan auth, dan registrasi dibuka lewat [`auth-service provision --allow-self-register`](../restforge-backend/auth-service/provision.md#registrasi-publik). Saat registrasi tertutup, link tersembunyi dan `register.html` hanya menampilkan `Registration is currently closed.` beserta link ke halaman login. Alur lengkapnya ada di [Registrasi Publik](../restforge-backend/auth-service/README.md#registrasi-publik).

### Plugin Assets

File vendor (jQuery, Select2, Flatpickr, Bootstrap, dll.) di-copy dari plugin ke folder output. Persisnya bervariasi per plugin.

## Behavior

Verb `generate`:

1. **Membaca** UDF dari path `--payload`
2. **Muat plugin** dari `appConfig.plugin` atau override `--plugin`
3. **Jalankan validator** UDF — error membatalkan generate
4. **Hitung daftar file** yang akan dihasilkan
5. **Cek folder output**:
   - Jika kosong atau tidak ada → buat folder dan tulis file
   - Jika ada file → error kecuali ditambah `--overwrite`
6. **Tulis** file HTML/JS/CSS/aset ke folder output
7. **Periksa artefak auth** (khusus plugin `vanilla-js-auth`/`vanilla-js-custom`) — lihat [Pemeriksaan Artefak Auth](#pemeriksaan-artefak-auth)
8. **Tampilkan ringkasan**: jumlah file written, daftar warning

### Pemeriksaan Artefak Auth

Halaman yang dihasilkan plugin `vanilla-js-auth` dan `vanilla-js-custom`
memanggil kontrak `window.Auth` untuk menyertakan header `Authorization` pada
request API. Setelah menulis output, `generate` memeriksa kelengkapan artefak
auth di folder output; bila ada yang belum terpasang (mis. `js/rfx_auth.js`),
warning ditampilkan beserta daftar file yang hilang dan perintah pemasangannya:

```
warning: generated pages depend on the `window.Auth` contract, but the following
auth artifacts are missing from the output:
  - js/rfx_auth.js
Run `npx restforge-designer auth --attach --project=<name>` to retrofit them.
```

Warning bersifat informatif: generate tetap sukses (exit code `0`) dan tidak ada
file yang diubah oleh pemeriksaan ini. Jalankan [`auth --attach`](./auth.md)
untuk memasang artefak yang hilang; setelah terpasang, warning tidak muncul lagi.

#### Aplikasi dengan auth-service

Aplikasi yang memakai layanan [`auth-service`](../restforge-backend/auth-service/README.md) tidak membutuhkan `js/rfx_auth.js`, sehingga `generate` tidak memperingatkan file tersebut. Aplikasi dianggap memakai auth-service bila plugin-nya `vanilla-js-auth` dan UDF memuat blok `auth` dengan `appCode` dan `authApiUrl` yang tidak kosong. Blok ini ditulis `payload migrate` saat `--auth-app-code` dan `--auth-api-url` diberikan.

| Kondisi UDF | Peringatan `js/rfx_auth.js` |
|-------------|--------------------------|
| `vanilla-js-auth` dengan blok `auth` (`appCode` dan `authApiUrl` terisi) | Tidak muncul |
| `vanilla-js-auth` tanpa blok `auth`, atau blok `auth` tidak lengkap | Muncul |
| `vanilla-js-custom`, dengan atau tanpa blok `auth` | Muncul |

Untuk aplikasi auth-service, artefak login plugin (`js/auth.js`, `login.html`, `js/login.js`, dan blok auth di `js/config.js`) tetap diperiksa. Bila ada yang hilang, petunjuknya meminta `generate` dijalankan tanpa `--scope=form`, bukan `auth --attach`.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Generate sukses |
| `1` | Validasi gagal, folder existing tanpa `--overwrite`, plugin tidak ditemukan, atau IO error |

## Menjalankan Hasil Generate

Setelah generate selesai:

```bash
cd ./apps/contact-mgmt-app
app-start.bat
```

Script `app-start.bat` menjalankan `npx serve -l <port>` di port yang ditentukan oleh `appConfig.port` di UDF. Default port: `8080`.

URL aplikasi: `http://localhost:<port>/index.html`.

## Use Case Umum

| Skenario | Command |
|----------|---------|
| First generate setelah scaffold project | `generate --payload=... --output=./apps/contact-mgmt-app` |
| Re-generate setelah edit UDF | `generate --payload=... --output=./apps/contact-mgmt-app --overwrite` |
| Iterasi cepat 1 page | `generate --payload=... --output=./apps/contact-mgmt-app --scope=form --page=contact --overwrite` |
| Bandingkan output plugin | `generate ... --plugin=vanilla-js-auth --output=./apps/contact-mgmt-app-auth` |

---

**Lihat juga**: [`README`](./README.md) · [`init.md`](./init.md) · [`auth.md`](./auth.md) · [`validate.md`](./validate.md) · [`preview.md`](./preview.md) · [`catalogs/udf/`](../../catalogs/udf/)
