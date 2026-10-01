# Field Auth Guard (`authGuard`)

Mengaktifkan proteksi autentikasi dan otorisasi pada endpoint hasil generate.
Bila blok ini aktif, generator turut menghasilkan middleware guard yang
memverifikasi JWT pada setiap request dan menegakkan permission per endpoint di
sisi server, tanpa perlu menulis middleware manual.

```json
{
    "authGuard": {
        "enabled": true,
        "appCode": "MY_APP",
        "publicPaths": ["/product/read"]
    }
}
```

## Aturan Konfigurasi

| Field | Tipe | Wajib | Keterangan |
|-------|------|:-----:|-----------|
| `enabled` | boolean | ✓ | `true` mengaktifkan guard. `false` atau blok tidak ada = output generate tanpa guard (backward compatible) |
| `appCode` | string | ✓ bila `enabled` | Kode aplikasi yang dicocokkan dengan klaim `app` pada token. Dapat dioverride saat runtime lewat env `AUTH_APP_CODE` |
| `publicPaths` | array of string | ✗ | Daftar path yang boleh diakses tanpa token (exact match, relatif terhadap prefix modul, wajib diawali `/`). `/health` dan `/info` selalu publik tanpa perlu dideklarasikan |

## Artefak yang Dihasilkan

Saat `endpoint create` dijalankan dengan blok `authGuard` aktif, generator
menulis file guard `src/plugins/<project>-auth-guard.js`. File ini digenerate
penuh setiap regenerate; jangan diedit manual. Kustomisasi dilakukan lewat
payload (`publicPaths`) atau lewat plugin custom terpisah.

Module loader memuat guard SEBELUM plugin custom `src/plugins/<project>-plugin.js`
(bila ada), sehingga plugin custom tetap berjalan normal dan tidak pernah
tertimpa oleh generator.

## Perilaku Runtime

Guard membaca token `Bearer` dari header `Authorization`, lalu:

| Kondisi | Hasil |
|---------|-------|
| Path terdaftar di `publicPaths`, atau `/health`/`/info` | Lolos tanpa token |
| Tanpa token pada path terjaga | `401` |
| Token tidak valid, expired, atau algoritma tidak sesuai | `401` |
| Klaim `app` tidak cocok dengan `appCode` | `403` (pengecualian: app `SYSTEM` dengan role `SUPER_ADMIN`) |
| Permission yang dituntut tidak ada di klaim `permissions` | `403` |
| Token valid dan permission cocok | Request diteruskan |

Token diharapkan memuat klaim `app` (kode aplikasi), `roles` (array), dan
`permissions` (array), seperti yang diterbitkan layanan auth RESTForge saat
login. Role `SUPER_ADMIN` dan `OWNER` melewati pemeriksaan permission.

### Pemetaan Permission per Endpoint

Permission yang dituntut berformat `<RESOURCE>_<AKSI>`. Nama resource diambil
dari nama endpoint (uppercase, `-` menjadi `_`), dan aksi diturunkan dari action
endpoint:

| Aksi | Action endpoint |
|------|-----------------|
| `CREATE` | `create`, `create-composite`, `import-upload`, `import-commit` |
| `READ` | `read`, `read-composite`, `datatables`, `lookup`, `first`, `aggregate`, `export`, `export-status`, `export-download`, `import-preview`, `import-status` |
| `UPDATE` | `update`, `update-composite`, `adjust`, `change-status`, `restore`, `upload`, `upload-url`, `upload-confirm` |
| `DELETE` | `delete`, `upload-delete`, `upload-cleanup` |

Contoh: `POST /product/datatables` menuntut permission `PRODUCT_READ`. Action di
luar tabel cukup membawa token valid tanpa permission spesifik.

## Konfigurasi Verifikasi Token (Env Runtime)

Mode verifikasi dipilih dari env server backend, diperiksa berurutan (yang
pertama terisi yang dipakai):

| Prioritas | Env | Mode |
|:---:|-----|------|
| 1 | `AUTH_JWKS_URL` | RS256; public key diambil dari endpoint JWKS layanan auth, di-cache dengan TTL `AUTH_JWKS_CACHE_TTL_MS` (default 10 menit) |
| 2 | `JWT_PUBLIC_KEY_PATH` | RS256; public key PEM dibaca dari file lokal |
| 3 | `JWT_SECRET` | HS256; shared secret yang sama dengan layanan auth |

Mode JWKS direkomendasikan untuk deployment lintas server karena backend tidak
perlu memegang secret apa pun milik layanan auth.

## Deklarasi pada Modul Multi-Endpoint

Blok `authGuard` dideklarasikan per file RDF, dan satu modul menghasilkan satu
file guard gabungan. Saat beberapa endpoint dalam modul yang sama
mendeklarasikannya:

- `publicPaths` digabungkan (union) dari seluruh endpoint.
- `appCode` memakai nilai dari endpoint yang terakhir di-generate; bila berbeda
  antar endpoint, generator menampilkan warning.
- Daftar resource terakumulasi setiap kali `endpoint create` dijalankan.

## Menonaktifkan Guard

Menghapus blok `authGuard` (atau men-set `enabled: false`) lalu regenerate
membuat generator berhenti memperbarui guard, tetapi file guard lama TIDAK
dihapus otomatis. Hapus `src/plugins/<project>-auth-guard.js` secara manual bila
proteksi ingin dilepas sepenuhnya, lalu restart server.

## Validasi

| Aturan | Perilaku |
|--------|----------|
| `authGuard` harus object | Error saat generate |
| `enabled` harus boolean | Error saat generate |
| `appCode` kosong saat `enabled: true` | Error saat generate |
| `publicPaths` bukan array of string berawalan `/` | Error saat generate |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`endpoint create`](../../commands/restforge-backend/endpoint/create.md) · [`README`](../../README.md)
