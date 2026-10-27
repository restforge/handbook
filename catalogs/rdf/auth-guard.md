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
| `readGrants` | object | ✗ | Endpoint lain yang boleh dibaca form ini tanpa permission endpoint tersebut, di luar yang sudah dibuka relasi FK. Lihat [Membaca Data Referensi dari Form Lain](#membaca-data-referensi-dari-form-lain) |

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

Token harus berasal dari layanan [auth-service](../../commands/restforge-backend/auth-service/README.md). Layanan itu mengisi klaim `app` (kode aplikasi), `roles` (array), dan `permissions` (array) saat login, dan guard memverifikasi tanda tangan token dengan kunci yang sama. Role `SUPER_ADMIN` dan `OWNER` melewati pemeriksaan permission.

Identitas pengguna dari token yang valid juga dipakai untuk mengisi kolom audit `created_by` dan `updated_by`. Nilainya diambil dari klaim `username`, atau klaim `sub` bila `username` kosong, dan didahulukan atas header `x-user-id`. Urutan lengkapnya ada di [Sumber Nilai `created_by` dan `updated_by`](./audit-columns.md#sumber-nilai-created_by-dan-updated_by).

Perintah [`auth-service link`](../../commands/restforge-backend/auth-service/link.md) memasang blok `authGuard` ke RDF project dan menulis `AUTH_APP_CODE` serta kunci verifikasi ke env aplikasi, sehingga keduanya tidak perlu ditulis manual. Selama env aplikasi berisi `AUTH_APP_CODE`, [`payload generate`](../../commands/restforge-backend/payload/generate.md) dan [`endpoint create`](../../commands/restforge-backend/endpoint/create.md) memasang blok `{ "enabled": true, "appCode": "<AUTH_APP_CODE>", "publicPaths": [] }` ke RDF yang belum punya key `authGuard`. Blok yang sudah ada tidak diubah, sehingga resource publik cukup diberi `"enabled": false` atau `publicPaths`. Tabel berawalan `auth.` dilewati.

### Pemetaan Permission per Endpoint

Permission yang dituntut berformat `<RESOURCE>_<AKSI>`. Nama resource diambil
dari nama endpoint (uppercase, `-` menjadi `_`), dan aksi diturunkan dari action
endpoint:

| Aksi | Action endpoint |
|------|-----------------|
| `CREATE` | `create`, `create-composite` |
| `READ` | `read`, `read-composite`, `datatables`, `lookup`, `first`, `aggregate` |
| `UPDATE` | `update`, `update-composite`, `adjust`, `restore`, `upload`, `upload-url`, `upload-confirm` |
| `DELETE` | `delete`, `upload-delete`, `upload-cleanup` |
| `EXPORT` | `export`, `export-status`, `export-download` |
| `IMPORT` | `import-upload`, `import-preview`, `import-commit`, `import-status` |
| `CHANGE_STATUS` | `change-status` |

Contoh: `POST /product/datatables` menuntut permission `PRODUCT_READ`, sedangkan
`POST /product/export` menuntut `PRODUCT_EXPORT`. Action di luar tabel cukup
membawa token valid tanpa permission spesifik.

Export, import, dan perubahan status workflow punya permission sendiri. Hak baca
atau ubah biasa tidak otomatis mencakup ketiganya.

## Membaca Data Referensi dari Form Lain

Form transaksi biasanya membaca data master, misalnya form Item memuat dropdown Category lewat `category/lookup`. Endpoint itu menuntut `CATEGORY_READ`, padahal permission tersebut juga menampilkan menu Category. Guard karena itu mengizinkan pembacaan data referensi atas nama form pemakainya, sehingga role yang hanya diberi permission `ITEM_*` tetap bisa memakai form Item tanpa ikut mendapat menu Category.

Request baca ke endpoint lain diizinkan bila memenuhi tiga syarat berikut.

- Request membawa header `X-Resource` berisi kode resource form, misalnya `X-Resource: ITEM`. Frontend hasil `restforge-designer` mengirim header ini otomatis dari setiap halaman.
- User memegang salah satu permission resource form tersebut, misalnya `ITEM_CREATE` saja. Role yang hanya boleh menambah item tetap bisa memuat dropdown Category.
- Endpoint dan action yang diminta terdaftar sebagai hak baca form tersebut.

Hak baca hanya berlaku untuk `lookup`, `read`, `first`, dan `datatables`. Request `create`, `update`, dan `delete` ke endpoint master tetap menuntut permission master. User yang memegang permission endpoint itu sendiri, misalnya `CATEGORY_READ`, tetap dilayani seperti biasa tanpa batasan di bawah.

### Hak Baca dari Relasi FK

Setiap relasi `belongsTo` di SDF tabel form otomatis membuka `lookup`, `read`, dan `first` ke tabel yang dirujuk. Relasi berikut di `schema/item.js` sudah cukup agar form Item bisa membaca Category, tanpa deklarasi apa pun di RDF.

```js
relations: {
  category: {
    type: 'belongsTo',
    localKey: 'category_id',
    references: 'category_id'
  }
}
```

Kolom yang bisa dibaca lewat hak ini hanya primary key tabel yang dirujuk dan kolom tampilannya, yaitu kolom yang sama dengan hasil `--expand-fk`. Untuk contoh di atas, kolomnya adalah `category_id` dan `category_name`.

`datatables` tidak ikut dibuka dari relasi FK, karena endpoint ini menyediakan pencarian dan paging atas seluruh isi tabel seperti halaman daftar di menu master.

### Deklarasi Manual dengan `readGrants`

`readGrants` ditulis di blok `authGuard` RDF form pemakai untuk dua kasus, yaitu `datatables` (misalnya modal pencarian Employee) dan tabel yang dibaca tanpa relasi FK. Contoh berikut ditulis di `payload/attendance-machine-log.json`.

```json
{
    "authGuard": {
        "enabled": true,
        "appCode": "HRM",
        "publicPaths": [],
        "readGrants": {
            "employee": {
                "actions": ["datatables"],
                "columns": ["employee_id", "employee_name", "department_id", "section_id"]
            }
        }
    }
}
```

| Field | Tipe | Wajib | Keterangan |
|-------|------|:-----:|-----------|
| key | string | ✓ | Nama endpoint yang dibaca, misalnya `employee` |
| `actions` | array of string | ✓ | Kombinasi `lookup`, `read`, `first`, dan `datatables` |
| `columns` | array of string | ✓ | Kolom yang boleh dikembalikan ke form ini |

Bila endpoint yang sama juga dibuka relasi FK, action dan kolom dari keduanya digabung. `readGrants` dengan begitu bisa menambah `datatables` atau kolom untuk relasi yang sudah ada.

### Batas Kolom

Request yang dilayani lewat hak baca form hanya menerima kolom dari daftar kolomnya. Request yang meminta kolom lain lewat `select`, termasuk di dalam ekspresi seperti `category_name || category_code AS text`, ditolak `403`. Response `read`, `first`, dan `datatables` tanpa `select` dipangkas ke daftar kolom tersebut.

Header `X-Resource` yang dipalsukan tidak menambah hak. User tetap harus memegang permission resource yang disebut di header, dan hanya bisa membaca endpoint, action, serta kolom yang terdaftar untuk resource itu.

Hak baca ditulis ke file guard saat `endpoint create` dijalankan untuk endpoint form, dan tidak menambah permission baru di auth-service. Matriks role di editor RBAC tetap hanya memuat permission menu.

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
| `readGrants` bukan object berkunci nama endpoint | Error saat generate |
| `actions` kosong atau memuat action selain `lookup`, `read`, `first`, `datatables` | Error saat generate |
| `columns` kosong atau memuat nama kolom tidak valid | Error saat generate |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`endpoint create`](../../commands/restforge-backend/endpoint/create.md) · [`README`](../../README.md)
