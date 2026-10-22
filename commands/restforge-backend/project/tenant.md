# `project tenant`

> Mengaktifkan atau menonaktifkan pemisahan data per tenant pada project RESTForge.

## Pattern

```
npx restforge project tenant --activate --project=<NAME> [OPTIONS]
npx restforge project tenant --deactivate --project=<NAME> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--activate` | Ya* | `false` | Mengaktifkan multi-tenant (`TENANT_MODE=multi`) |
| `--deactivate` | Ya* | `false` | Menonaktifkan multi-tenant (`TENANT_MODE=single`) tanpa menghapus kolom maupun data |
| `--project <NAME>` | Tidak | — | Project yang endpoint-nya dibuat ulang. Tanpa flag ini, endpoint tidak dibuat ulang |
| `--name <NAME>` | Tidak | — | Alias dari `--project` |
| `--schema-path <DIR>` | Tidak | `./schema` | Folder SDF yang diperiksa |
| `--payload-path <DIR>` | Tidak | `./payload` | Folder RDF yang diperbarui |
| `--config <FILE>` | Tidak | `config/db-connection.env` | File config yang dibaca dan ditulis key `TENANT_*`-nya |
| `--default-tenant <ID\|CODE>` | Tidak | — | Tenant untuk baris lama yang belum punya tenant |
| `--dry-run` | Tidak | `false` | Menampilkan rencana tanpa menulis file atau mengubah database |
| `--force` | Tidak | `false` | Menjalankan ulang meskipun mode sudah sesuai, termasuk menulis ulang SDF dan `requestScope` |

*Pilih salah satu dari `--activate` atau `--deactivate`. Keduanya tidak boleh dipakai bersamaan.

## Prasyarat

- Project sudah memakai autentikasi, yaitu [`project auth`](./auth.md) atau [`auth-service`](../auth-service/README.md). Filter tenant mengambil nilai tenant dari token user yang login.
- Database aktif dan terkonfigurasi di `--config`.

## Aktivasi

`--activate` mengubah project yang sudah berjalan menjadi multi-tenant. Perintah ini aman dijalankan ulang, karena langkah yang sudah selesai dilewati.

1. Config mendapat `TENANT_MODE=multi`. `TENANT_COLUMN=tenant_id` dan `TENANT_BYPASS_ROLES` kosong ditambahkan bila key itu belum ada.
2. SDF tabel bisnis yang belum punya field `tenant_id` mendapat field `tenant_id` dan index-nya. SDF lama diarsipkan ke `.restforge/archive/` sebelum ditulis ulang.
3. Unique pada tabel bisnis diubah menjadi unique per tenant, misalnya `UNIQUE (category_code)` menjadi `UNIQUE (tenant_id, category_code)`, sehingga setiap tenant boleh memakai kode yang sama.
4. RDF tabel bisnis mendapat blok `requestScope` yang memfilter data berdasarkan `tenant_id` milik user yang login. Kolom `tenant_id` ikut masuk `fieldName`, dan query JOIN hasil generate ikut memilih kolom itu.
5. Database mendapat kolom `tenant_id`. Baris lama diisi tenant default, lalu kolom dibuat NOT NULL dan diberi index. Unique di database diubah mengikuti SDF.
6. Endpoint project `--project` yang memakai tabel tersebut dibuat ulang.

Rencana setiap langkah tampil di output, termasuk perubahan unique per tabel.

```text
SDF mutation plan:
  [+tenant_id +index] category → schema\category.js
    category: UNIQUE (category_code) -> UNIQUE (tenant_id, category_code)
```

### Tabel yang Dicakup

Tabel registry tenant dan tabel token refresh milik auth extension tidak diberi `tenant_id`. Pada project auth-service, seluruh tabel di schema `auth` dilewati. Tabel user auth extension sudah punya `tenant_id`, sehingga hanya mendapat `requestScope` dan unique per tenant.

Beberapa unique tetap berlaku untuk seluruh tabel dan ditandai `stays table-wide` di rencana.

- Unique `username` pada tabel user, karena login mencari user tanpa tenant.
- Unique pada kolom foreign key, atau pada kolom yang dirujuk foreign key tabel lain.
- Unique pada tabel soft delete.

### Tenant untuk Baris Lama

Baris yang sudah ada sebelum aktivasi diisi tenant dari `--default-tenant`, yang menerima id atau code tenant. Tanpa flag ini, auth extension memakai tenant pertama di registry, atau membuat tenant `DEFAULT` bila registry masih kosong. Project auth-service memakai tenant berkode `DEFAULT` milik app tersebut, yang dibuat `auth-service provision`. Tabel kosong tidak membutuhkan tenant default.

## Project Auth Service

Project yang config-nya berisi `AUTH_APP_CODE` diproses dengan aturan auth-service.

- Hanya PostgreSQL yang didukung.
- `TENANT_COLUMN` harus `tenant_id`. Nilai lain menghentikan aktivasi sebelum ada file yang ditulis.
- Tenant diambil dari registry layanan auth milik app tersebut. Bila app atau tenant default tidak ditemukan, aktivasi berhenti sebelum menulis apa pun.
- Router endpoint tidak diberi `requireAuth`, karena `authGuard` project sudah mengisi data user dari token.

## Konfigurasi

| Key | Keterangan |
|-----|-----------|
| `TENANT_MODE` | `multi` saat aktif, `single` saat nonaktif |
| `TENANT_COLUMN` | Nama kolom tenant, default `tenant_id` |
| `TENANT_BYPASS_ROLES` | Role yang boleh membaca dan mengubah data semua tenant, dipisah koma. Default kosong |

Daftar bypass role ditulis ke endpoint saat aktivasi. Perubahan `TENANT_BYPASS_ROLES` baru berlaku setelah `--deactivate` lalu `--activate` dijalankan lagi. Role bypass tidak bisa membuat data baru, karena token-nya tidak membawa tenant.

## Tabel Baru setelah Aktivasi

Tabel yang ditambahkan setelah aktivasi tidak membutuhkan `project tenant --activate` lagi. Selama config berisi `TENANT_MODE=multi`, [`schema migrate`](../schema/migrate.md#project-multi-tenant) membuat tabel baru dengan kolom `tenant_id` dan unique per tenant, lalu [`payload generate`](../payload/generate.md#project-multi-tenant) menambahkan `requestScope` ke RDF-nya. Endpoint tabel lain tidak dibuat ulang.

```bat
npx restforge schema migrate --schema-path=./schema/supplier.js
npx restforge payload generate --table=supplier
npx restforge endpoint create --project=myapp --name=supplier --payload=supplier.json
```

Aktivasi ulang hanya diperlukan untuk tabel yang sudah ada di database tanpa kolom `tenant_id`, karena baris di dalamnya perlu diisi tenant. `schema migrate` menampilkan peringatan untuk tabel seperti ini.

Setelah aktivasi, dan setiap kali tabel, RDF, atau processor ditambahkan, [`project verify`](./verify.md) memeriksa ulang seluruh aturan ini, termasuk SQL processor dan dashboard yang ditulis sendiri.

## Deaktivasi

`--deactivate` menulis `TENANT_MODE=single`, melepas `requestScope` tenant dari RDF, lalu membuat ulang endpoint `--project`. Kolom `tenant_id` beserta isinya, unique per tenant, `TENANT_COLUMN`, dan `TENANT_BYPASS_ROLES` tetap ada. Aktivasi berikutnya tidak perlu mengisi tenant untuk baris lama lagi.

## Contoh

```bat
:: Pratinjau rencana aktivasi
npx restforge project tenant --activate --project=myapp --dry-run

:: Aktivasi dengan tenant tertentu untuk baris lama
npx restforge project tenant --activate --project=myapp --default-tenant=DEFAULT

:: Menonaktifkan multi-tenant
npx restforge project tenant --deactivate --project=myapp
```

Setelah aktivasi atau deaktivasi, server yang sedang berjalan perlu dijalankan ulang.

## Catatan

- SQLite tidak bisa mengubah kolom yang sudah ada, sehingga kolom `tenant_id` pada tabel lama tetap boleh kosong dan unique lama di database tidak diubah. Aktivasi menampilkan peringatan untuk keduanya. Tabel baru yang dibuat `schema migrate` setelah aktivasi tidak terkena batasan ini.
- Query JOIN yang sudah diedit manual tidak ditimpa. Aktivasi menampilkan peringatan yang menyebut nama file, lalu kolom `tenant_id` perlu ditambahkan ke query itu secara manual.
- Endpoint yang dibuat ulang menimpa kustomisasi manual pada file endpoint. Jalankan `--dry-run` lebih dulu untuk melihat endpoint yang terdampak.

---

**Lihat juga**: [`project/`](./) · [`project verify`](./verify.md) · [`schema migrate`](../schema/migrate.md) · [`payload generate`](../payload/generate.md) · [`commands/`](../) · [`README`](../../../README.md)
