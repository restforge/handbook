# `project verify`

> Memeriksa konsistensi seluruh project, terutama pemisahan data per tenant, sebelum deploy atau di pipeline CI.

## Pattern

```
npx restforge project verify --project=<NAME> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | — | Project yang diperiksa |
| `--name <NAME>` | Tidak | — | Alias dari `--project` |
| `--schema-path <DIR>` | Tidak | `./schema` | Folder SDF |
| `--payload-path <DIR>` | Tidak | `./payload` | Folder RDF |
| `--config <FILE>` | Tidak | — | Mengaktifkan perbandingan SDF dan RDF dengan database (VAL004). Tanpa flag ini, perintah hanya membaca file |
| `--strict` | Tidak | `false` | Warning dihitung sebagai error |
| `--json` | Tidak | `false` | Output JSON untuk CI dan MCP |

Nilai `TENANT_MODE` dan `TENANT_COLUMN` dibaca dari `--config`. Tanpa flag itu, perintah membaca config default dari [`config set-default`](../config/set-default.md), lalu `config/db-connection.env`. Membaca file config tidak membuka koneksi database.

## Cara Kerja

`project tenant --activate` memasang pemisahan data sekali, yaitu pada file yang ada saat perintah itu dijalankan. Tabel baru, RDF yang diedit manual, dan SQL yang ditulis sendiri di processor atau dashboard tidak pernah diperiksa ulang. `project verify` menjalankan ulang seluruh aturan itu kapan saja, tanpa mengubah file project.

Satu-satunya file yang ditulis adalah lock file `.restforge/verify-lock.json`, dan hanya bila tidak ada error. Lock file mencatat hash setiap file pada verifikasi terakhir yang lolos, sehingga verifikasi berikutnya bisa menunjukkan file yang berubah sejak saat itu.

Setiap temuan punya kode tetap dan salah satu dari tiga tingkat berikut.

| Tingkat | Pengaruh |
|---------|----------|
| Error | Exit code 1 dan lock file tidak ditulis |
| Warning | Hanya dilaporkan. Dengan `--strict`, warning diperlakukan seperti error |
| Info | Keterangan, tidak memengaruhi exit code |

## Daftar Pemeriksaan

### Pemisahan Data pada SDF, RDF, dan UDF

Pemeriksaan ini hanya berjalan saat `TENANT_MODE=multi`. Tabel yang dikecualikan sama dengan aturan [`project tenant`](./tenant.md#tabel-yang-dicakup).

| Kode | Yang diperiksa | Tingkat |
|------|----------------|---------|
| TEN001 | Setiap tabel di SDF punya kolom tenant | Error |
| TEN002 | Unique satu kolom pada tabel ber-tenant sudah berbentuk unique per tenant, kecuali unique yang memang berlaku untuk seluruh tabel | Error |
| TEN010 | RDF tabel ber-tenant memuat `requestScope` dengan kolom tenant dan sumber `req.user.<kolom>` | Error |
| TEN011 | Query JOIN di RDF memilih kolom tenant milik tabel utama | Error |
| TEN012 | RDF tabel user auth extension melindungi kolom `role` lewat `requestScope.protectedColumns` | Error |
| TEN013 | Endpoint tabel ber-tenant sudah dibuat ulang dengan filter tenant, termasuk endpoint yang dibuat sebelum aktivasi | Error |
| TEN014 | `tenantForeignKeys` di RDF ber-`requestScope` sesuai relasi SDF, sehingga setiap foreign key ke tabel ber-tenant diperiksa tenant-nya (lihat [Foreign Key ke Data Tenant Lain](./tenant.md#foreign-key-ke-data-tenant-lain)) | Error |
| TEN020 | Sumber data UDF bertipe `api`, misalnya field select, cascade `dependsOn`, dan lookup grid detail, hanya memakai endpoint yang lolos TEN010 | Error |

### Pemisahan Data pada SQL yang Ditulis Sendiri

SQL di bagian ini ditulis developer, sehingga pemeriksaannya bersifat heuristik dan bisa menghasilkan false positive. Karena itu, sebagian besar tingkatnya masih warning.

| Kode | Yang diperiksa | Tingkat |
|------|----------------|---------|
| TEN030 | SQL `processor[].sql` di RDF menyaring kolom tenant. SELECT, UPDATE, dan DELETE menyebutnya di WHERE, sedangkan INSERT mengisinya | Warning |
| TEN031 | Query `executeQuery` dan `executeTransaction` di file processor `src/modules/<project>/processor/` memenuhi aturan TEN030, dan nilai tenant-nya berasal dari `req.user.<kolom>` | Warning |
| TEN032 | SQL widget dashboard menyaring kolom tenant | Warning |
| TEN033 | SQL yang tidak bisa dibaca, misalnya hasil penggabungan string atau dibentuk di helper function, dilaporkan sebagai "could not be verified" dan tidak dianggap lolos | Warning |
| TEN034 | Anotasi pengecualian memuat alasan | Error |

Query yang memang membaca data lintas tenant diberi anotasi pengecualian beserta alasannya. Di file processor, anotasi ditulis di baris tepat sebelum pemanggilan.

```js
// @restforge-tenant-ignore: laporan agregat lintas tenant khusus SUPER_ADMIN
const rows = await db.executeQuery(sql, params);
```

Di file SQL atau SQL inline, anotasi ditulis sebagai komentar SQL.

```sql
-- @restforge-tenant-ignore: laporan agregat lintas tenant khusus SUPER_ADMIN
SELECT tenant_id, count(*) AS total FROM sales_order GROUP BY tenant_id
```

Setiap pengecualian tercantum di output dan di lock file agar bisa diaudit. Processor login, register, dan refresh token yang dibuat [`project auth`](./auth.md) sudah membawa anotasi ini, karena query-nya berjalan sebelum tenant user diketahui.

### Konsistensi Definisi

| Kode | Yang diperiksa | Tingkat |
|------|----------------|---------|
| VAL001 | SDF lolos [`schema validate`](../schema/validate.md) mode file | Error |
| VAL002 | Setiap RDF merujuk tabel dan kolom yang ada di SDF. Kolom JOIN dianggap sah bila dipilih salah satu query RDF | Error |
| VAL003 | UDF utama di `frontend/payload/` lolos validasi `restforge-designer` | Error |
| VAL004 | Dengan `--config`, SDF dan RDF sesuai dengan struktur tabel di database | Error |

### Perubahan sejak Verifikasi Terakhir

Pemeriksaan ini membandingkan project dengan lock file. Tanpa lock file, misalnya di CI atau di clone baru, pemeriksaan ini dilewati.

| Kode | Yang diperiksa | Tingkat |
|------|----------------|---------|
| INT001 | `TENANT_MODE` atau `TENANT_COLUMN` berbeda dengan nilai di lock file | Info |
| INT002 | File SDF, RDF, SQL, UDF, dan processor yang berubah, bertambah, atau hilang | Info |
| INT003 | File hasil generate di `src/modules/` dan `src/models/` berubah, padahal RDF dan SDF sumbernya tidak berubah | Error |

INT003 menandai edit manual pada output generate. Perbaikannya adalah memindahkan perubahan ke RDF lalu membuat ulang endpoint, atau membuat ulang endpoint untuk mengembalikan isi aslinya. File processor tidak termasuk, karena memang ditulis developer. INT003 dilewati bila versi platform berubah sejak verifikasi terakhir, karena template generate ikut berubah.

## Output

Output plain mengelompokkan temuan per file.

```text
Verify myapp (TENANT_MODE=multi, tenant column: tenant_id)

schema/supplier.js
  ERROR   TEN001  Table supplier has no tenant_id column. Run `npx restforge schema migrate` for a new table, or `npx restforge project tenant --activate` for a table that already holds data.

src/modules/myapp/processor/report/stock.js
  WARNING TEN031  Line 24: SELECT on inventory.stock does not filter tenant_id.

Skipped checks:
  VAL004: --config not provided

Changed since last successful verification: 3 files
  payload/supplier.json
  schema/supplier.js
  src/modules/myapp/processor/report/stock.js
Result: 1 error, 1 warning. Lock file not updated.
```

Dengan `--json`, setiap temuan memuat `code`, `level`, `file`, `line`, dan `message`. Laporan JSON juga memuat `summary`, `exitCode`, daftar pemeriksaan yang dilewati di `skipped`, pengecualian di `ignores`, dan status lock file di `lock`.

## Lock File

Lock file adalah catatan lokal di mesin developer dan tidak di-commit. Saat pertama kali ditulis, `.restforge/verify-lock.json` ditambahkan ke `.gitignore` project. Satu folder bisa berisi beberapa project, misalnya layanan auth dan aplikasinya, sehingga lock file menyimpan entri per project.

```json
{
  "version": 1,
  "projects": {
    "myapp": {
      "verifiedAt": "2026-10-08T10:15:00.000Z",
      "platformVersion": "5.8.13",
      "tenantMode": "multi",
      "tenantColumn": "tenant_id",
      "sources": {
        "schema/supplier.js": "sha256:9f2c...",
        "payload/supplier.json": "sha256:41ab..."
      },
      "generated": {
        "src/modules/myapp/supplier.js": {
          "hash": "sha256:c3e1...",
          "from": "payload/supplier.json",
          "schema": "schema/supplier.js"
        }
      },
      "ignores": [
        { "file": "src/modules/myapp/processor/report/stock.js", "line": 23, "reason": "laporan agregat lintas tenant khusus SUPER_ADMIN" }
      ]
    }
  }
}
```

Hash hanya menunjukkan bahwa file berubah. Hasil verifikasi tetap ditentukan oleh aturan yang dijalankan ulang. Server tidak membaca lock file, sehingga `serve` tetap berjalan walaupun lock file tidak ada atau sudah tidak sesuai.

## Contoh

```bat
:: Pemeriksaan file saja
npx restforge project verify --project=myapp

:: Pipeline CI: warning ikut menggagalkan, output JSON
npx restforge project verify --project=myapp --strict --json

:: Termasuk perbandingan dengan database
npx restforge project verify --project=myapp --config=config/db-connection.env
```

## Catatan

- Kafka consumer tidak termasuk cakupan verifikasi.
- Processor yang mengakses database lewat query builder atau model, bukan `executeQuery` dan `executeTransaction`, tidak diperiksa.
- Perbandingan dengan database (VAL004) memakai pemeriksaan yang sama dengan `schema validate --config` dan `payload validate`. Pesan temuan menyebut perintah yang menampilkan detailnya.

---

**Lihat juga**: [`project/`](./) · [`project tenant`](./tenant.md) · [`schema validate`](../schema/validate.md) · [`commands/`](../) · [`README`](../../../README.md)
