# `npx restforge-designer auth`

> Memasang (`--create`), me-retrofit (`--attach`), atau mencabut (`--remove`) kerangka auth frontend pada project frontend RESTForge yang sudah ada. Ketiga mode saling eksklusif: tepat satu yang dipakai per invocation.

## `--create`

Memasang artefak auth (`login.html`, `signup.html`, `js/rfx_auth.js`) dan menginjeksi guard ke halaman app yang sudah ada.

### Sintaks

```bash
npx restforge-designer auth --create --project=<NAME> [OPTIONS]
```

### Flag

| Flag | Tipe | Wajib | Default | Keterangan |
|------|------|:-----:|---------|-----------|
| `--create` | flag | ✓ | — | Trigger instalasi (mutually exclusive dengan `--remove`) |
| `--project <NAME>` | string | ✓ | — | Nama project frontend; target app dir = `<frontend-path>/<project>` |
| `--frontend-path <PATH>` | string | ✗ | `./frontend/apps` | Root folder frontend apps |
| `--api-base-url <URL>` | string | ✗ | dari `payload/app-config.json` | URL base API; fallback ke `http://127.0.0.1:3000/api/<project>` bila tidak ditemukan |
| `--overwrite` | flag | ✗ | `false` | Timpa file auth yang ada (archive backup dibuat) |

### Apa yang Dikerjakan

1. Render `login.html`, `signup.html`, `js/rfx_auth.js` dari template embed (tanpa Google Sign-In)
2. Tulis 3 artefak ke `<frontend-path>/<project>/`
3. Inject guard `<script src="js/rfx_auth.js">` ke semua `*.html` existing di target dir (kecuali `login.html` dan `signup.html`)
4. Tulis marker `embeddedAuth` ke `payload/app-config.json` (non-destruktif; key lain tidak disentuh)

**Idempoten** — aman dijalankan ulang. File dilewati bila sudah ada, kecuali `--overwrite` diberikan.

**Perilaku saat tidak ada halaman app** — inject guard dilewati dengan warning; guard diinjeksi otomatis saat `generate` membuat halaman baru.

### Contoh

```bash
npx restforge-designer auth --create --project=myapp
npx restforge-designer auth --create --project=myapp --api-base-url=http://localhost:3032/api
npx restforge-designer auth --create --project=myapp --overwrite
```

---

## `--attach`

Me-retrofit kerangka auth lengkap ke project frontend yang halamannya sudah
di-generate, tanpa menyentuh page files dan tanpa merusak kustomisasi. Dipakai
saat aplikasi berjalan ingin dinyalakan auth-nya belakangan, termasuk saat
`generate` menampilkan warning artefak auth hilang.

### Sintaks

```bash
npx restforge-designer auth --attach --project=<NAME> [OPTIONS]
```

### Flag

| Flag | Tipe | Wajib | Default | Keterangan |
|------|------|:-----:|---------|-----------|
| `--attach` | flag | ✓ | — | Trigger retrofit (mutually exclusive dengan `--create`/`--remove`) |
| `--project <NAME>` | string | ✓ | — | Nama project frontend; target app dir = `<frontend-path>/<project>` |
| `--frontend-path <PATH>` | string | ✗ | `./frontend/apps` | Root folder frontend apps |
| `--api-base-url <URL>` | string | ✗ | dari `payload/app-config.json` | URL base API bila belum terkonfigurasi |
| `--overwrite` | flag | ✗ | `false` | Timpa file auth yang ada (archive backup dibuat) |

### Apa yang Dikerjakan

Retrofit bekerja dalam dua lapisan sesuai kondisi project:

1. **Kerangka `window.Auth` (selalu)** — pasang `js/rfx_auth.js`, inject
   `<script src="js/rfx_auth.js">` ke halaman existing (kecuali halaman login),
   dan tulis marker `embeddedAuth` ke `payload/app-config.json`. Halaman hasil
   generate memanggil kontrak ini untuk menyertakan header `Authorization` pada
   setiap request API.
2. **Artefak login plugin (bila aktif)** — bila payload UDF project memuat blok
   `auth` dan plugin-nya `vanilla-js-auth` atau `vanilla-js-custom`, render juga
   `js/auth.js`, `login.html`, dan `js/login.js` versi plugin, lalu injeksikan
   blok `appCode`/`authBaseUrl` ke `js/config.js` secara idempoten (file config
   yang sudah ada tidak ditimpa, hanya ditambah blok bertanda marker). Pada mode
   ini `login.html`/`signup.html` versi `rfx_auth` tidak ditulis; login memakai
   versi plugin, dan storage key `rfx_auth` otomatis diselaraskan dengan login
   plugin sehingga token hasil login terbaca oleh `Auth.authHeaders()`.

**Idempoten** — aman dijalankan berulang. File yang sudah ada dilewati kecuali
`--overwrite` diberikan (archive backup dibuat lebih dulu); injeksi script tag
dan blok config tidak pernah ganda.

**Page files tidak disentuh** — seluruh halaman aplikasi dan kustomisasinya
dibiarkan utuh.

### Contoh

```bash
npx restforge-designer auth --attach --project=myapp
npx restforge-designer auth --attach --project=myapp --frontend-path=frontend/apps
npx restforge-designer auth --attach --project=myapp --overwrite
```

### `--attach` atau `--create`?

| Kebutuhan | Mode |
|-----------|------|
| Menambah overlay login/signup `rfx_auth` standalone ke app apa pun | `--create` |
| Melengkapi kerangka auth project ber-plugin `vanilla-js-auth`/`vanilla-js-custom` yang halamannya sudah di-generate | `--attach` |

---

## `--remove`

Mencabut artefak auth, melepas guard dari halaman app, dan menghapus marker dari `payload/app-config.json`.

### Sintaks

```bash
npx restforge-designer auth --remove --project=<NAME> [OPTIONS]
```

### Flag

| Flag | Tipe | Wajib | Default | Keterangan |
|------|------|:-----:|---------|-----------|
| `--remove` | flag | ✓ | — | Trigger pencabutan (mutually exclusive dengan `--create`) |
| `--project <NAME>` | string | ✓ | — | Nama project frontend target |
| `--frontend-path <PATH>` | string | ✗ | `./frontend/apps` | Root folder frontend apps |
| `--force` | flag | ✗ | `false` | Skip konfirmasi y/N |

### Apa yang Dikerjakan

1. Deteksi apakah auth terpasang (cek file + marker `embeddedAuth`)
2. Konfirmasi y/N (default N) sebelum menghapus; non-TTY abort kecuali `--force`
3. Hapus `login.html`, `signup.html`, `js/rfx_auth.js`
4. Lepas guard `<script src="js/rfx_auth.js">` dari semua `*.html` (konten halaman lain utuh)
5. Hapus key `embeddedAuth` dari `payload/app-config.json` (key lain dipertahankan)

**Idempoten** — no-op bila auth tidak terpasang; menjalankan `--remove` dua kali aman.

### Contoh

```bash
npx restforge-designer auth --remove --project=myapp
npx restforge-designer auth --remove --project=myapp --force
```

---

## Kontrak `window.Auth`

`js/rfx_auth.js` mengekspos objek global `window.Auth` yang dipakai halaman
hasil generate. Kontrak ini juga menjadi acuan bila aplikasi ingin memanggil
API secara manual dari script kustom.

| Anggota | Keterangan |
|---------|-----------|
| `Auth.getToken()` | Access token aktif, atau `null` bila belum login |
| `Auth.getRefreshToken()` | Refresh token aktif |
| `Auth.getUser()` | Objek user hasil login |
| `Auth.setSession(data)` | Simpan sesi (token + user) ke storage |
| `Auth.clearSession()` | Hapus seluruh sesi dari storage |
| `Auth.authHeaders(extra)` | Objek header untuk request API (lihat semantik di bawah) |
| `Auth.logout()` | Akhiri sesi lalu arahkan ke halaman login |
| `Auth.API_BASE_URL` | URL base API yang dipakai |

### Semantik `authHeaders(extra)`

`extra` (opsional) adalah objek header tambahan milik pemanggil dan menjadi
dasar hasil: `Authorization: Bearer <token>` DITAMBAHKAN ke dalamnya bila token
ada; tanpa token, hasilnya berisi `extra` apa adanya. Selalu oper header
tambahan (mis. `Content-Type`, header custom) lewat parameter `extra` agar tidak
hilang:

```javascript
fetch(url, {
    method: "POST",
    headers: Auth.authHeaders({ "Content-Type": "application/json" }),
    body: JSON.stringify(payload)
});
```

Sesi disimpan di `localStorage`. Pada pemasangan via `--create` key storage
berprefix nama project; pada `--attach` dengan artefak login plugin, key
mengikuti login plugin sehingga kedua sistem membaca sesi yang sama.

---

## Catatan

Google Sign-In dan paket `@restforgejs/auth` (auth+RBAC backend) tidak dicakup
command ini. `--create`/`--remove` mengelola overlay `rfx_auth` embedded;
`--attach` juga merender artefak login plugin `vanilla-js-auth`/
`vanilla-js-custom` bila payload project mengaktifkan blok `auth`. Proteksi
sisi server dikonfigurasi terpisah lewat blok payload backend
[`authGuard`](../../catalogs/rdf/auth-guard.md).

---

**Lihat juga**: [`restforge-frontend/`](./) · [`commands/`](../) · [`README`](../../README.md)
