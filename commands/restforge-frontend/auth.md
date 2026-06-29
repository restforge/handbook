# `npx restforge-designer auth`

> Memasang atau mencabut embedded auth frontend (`rfx_auth`) ke project frontend RESTForge yang sudah ada, tanpa bergantung pada plugin `vanilla-js-auth` atau paket `@restforgejs/auth`.

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

## Catatan

Google Sign-In, plugin `vanilla-js-auth`, dan paket `@restforgejs/auth` (auth+RBAC) tidak dicakup — `auth --create`/`--remove` hanya mengelola `rfx_auth` embedded.

---

**Lihat juga**: [`restforge-frontend/`](./) · [`commands/`](../) · [`README`](../../README.md)
