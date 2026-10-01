# `payload validate`

> Memvalidasi file payload existing terhadap struktur tabel di database. Memberikan verdict pass/fail (OK / DRIFT / ERROR) tanpa detail per-column.

## Pattern

```
npx restforge payload validate --config=<FILE> [--table=<NAME>]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak | semua | Validasi hanya satu tabel spesifik |

## Logika Validasi

`payload validate` memakai rujukan tunggal dengan
`endpoint create` dan `payload diff`/`sync`: comparator audit-column-aware
`compareSchemaStrict`. Verdict yang dilaporkan:

| Status | Arti |
|--------|------|
| `[OK]` | Schema payload aligned dengan database |
| `[DRIFT]` | Ada perbedaan kolom, tipe, atau audit columns missing |
| `[ERROR]` | Tabel tidak ditemukan di database, validasi struktur RDF gagal, atau query SQL gagal dijalankan (lihat [Query SQL yang Gagal](#query-sql-yang-gagal)) |

Validasi struktur RDF dijalankan lebih dulu daripada perbandingan ke database. Bila sebuah file gagal di tahap itu, file langsung berstatus `[ERROR]` dengan ringkasan berisi pesan error struktur, perbandingan schema untuk file tersebut dilewati, dan exit code command menjadi `1`. Salah satu pemicunya adalah entri `datatablesWhere` yang tidak valid, dengan aturan lengkap di [Aturan `datatablesWhere`](../../../catalogs/rdf/validation-rules.md#aturan-datatableswhere). Pemicu lain adalah kolom JOIN di `datatablesQuery` yang memakai nama kolom tabel utama, dengan aturan di [Aturan Kolom JOIN di `datatablesQuery`](../../../catalogs/rdf/validation-rules.md#aturan-kolom-join-di-datatablesquery).

> **Peringatan upgrade:** Payload lapangan yang menulis `datatablesWhere` dengan prefiks alias tabel (misalnya `a.supplier_code`) sebelumnya lolos tanpa keluhan walau entri itu tidak pernah berfungsi saat pencarian. Payload semacam itu sekarang berstatus `[ERROR]` di sini dan menghentikan [`endpoint create`](../endpoint/create.md#validasi-schema-database). Perbaikannya adalah menyunting entri menjadi nama kolom hasil SELECT yang disebutkan pesan error.

Kategori drift yang dideteksi:

| Kategori | Deskripsi |
|----------|-----------|
| Payload field missing from database | `fieldName` di payload tidak ada di DB |
| Type change | Tipe data di `fieldValidation` payload beda dengan DB |
| New database column | DB punya kolom yang tidak ada di payload (non-audit) |
| Audit column required but missing | `auditColumns` aktif (default `true`) tapi DB tidak punya kolom audit standar |

Kolom yang sengaja dihapus dari `fieldName` tidak dilaporkan sebagai New database column. RESTForge mengenali kolom tersebut dari daftar kolom di folder `payload/.meta/` (lihat [Kolom yang Dihapus dari `fieldName`](./generate.md#kolom-yang-dihapus-dari-fieldname)). Pada RDF lama yang belum punya daftar kolom, kolom itu tetap dilaporkan sampai `payload sync` dijalankan sekali.

## Query SQL yang Gagal

`payload validate` menjalankan query pada `viewQuery`, `datatablesQuery`, dan `exportQuery` ke database, termasuk query di file SQL yang dirujuk lewat `file:`. Query yang gagal selalu ditampilkan di bawah verdict beserta pesan database:

```
  [ERROR] users.json (users)
         Query check failed for datatablesQuery
         Query check failed against the database:
           [!] datatablesQuery: 42P01: relation "userz" does not exist
```

Status file bergantung pada penyebab kegagalannya:

- Query gagal karena memilih kolom yang sudah di-drop dari tabel. File berstatus `[DRIFT]` dan kolom itu dilaporkan sebagai Payload field missing from database. `payload sync` menyelesaikan kondisi ini (lihat [Kolom yang Di-drop dari Tabel](./sync.md#kolom-yang-di-drop-dari-tabel)).
- Query gagal tanpa drift yang menjelaskannya, misalnya karena salah ketik nama tabel. File berstatus `[ERROR]`. Perbaiki query di RDF atau di file SQL yang dirujuk, karena `payload sync` tidak mengubah query tersebut.

Kolom yang di-drop dikenali dari daftar kolom di folder `payload/.meta/`. Pada RDF lama yang belum punya daftar kolom, kolom itu tidak dilaporkan sebagai drift, sehingga file berstatus `[ERROR]`.

## Output Format

```
============================================================
SCHEMA VALIDATION - Payload vs Database
============================================================

  Payload directory : <path>
  Files to validate : <count>

------------------------------------------------------------

  [OK]    users.json (users)
  [DRIFT] visitors.json (visitors)
         4 audit column(s) required but missing
  [ERROR] legacy.json (legacy)
         Table "legacy" not found in database

------------------------------------------------------------

  Summary:
    Total  : 3 payload(s)
    OK     : 1
    Drift  : 1
    Error  : 1

  Advisories (informational, not errors):
    [i] visitors: has 1 foreign key(s) not surfaced as JOIN
        -> npx restforge payload sync --table=visitors --expand-fk

  Use 'npx restforge payload diff' for detailed comparison.
  Use 'npx restforge payload sync' to update payload files automatically.
```

## Saran Tindak Lanjut

Selain verdict, `payload validate` dapat menampilkan bagian **Advisories** berisi
saran tindak lanjut. Advisory bersifat informational, **bukan** error: tidak
memengaruhi verdict (`OK`/`DRIFT`/`ERROR`) maupun exit code. Bagian ini hanya
muncul bila ada saran yang relevan, dan dilewati untuk file ber-status `[ERROR]`
(tabel tidak ada di database).

| Advisory | Pemicu | Saran |
|----------|--------|-------|
| Foreign key belum di-surface sebagai JOIN | Tabel punya foreign key dan payload belum memakai JOIN (tidak ada `viewQuery`) | `npx restforge payload sync --table=<NAME> --expand-fk` |
| `is_active` ada tetapi `defaultScope` belum lengkap | Kolom `is_active` ada di `fieldName` tetapi `defaultScope` belum memuat `is_active` di `lookup` dan `read` | `npx restforge payload sync --table=<NAME>` |

Advisory foreign key dilewati bila payload sudah memakai konfigurasi JOIN (lihat
[`payload sync --expand-fk`](./sync.md#ekspansi-foreign-key---expand-fk)). Advisory
scope dilewati bila `defaultScope` sudah lengkap (lihat [Default Scope `is_active`
built-in](./sync.md#default-scope-is_active-built-in)).

## Exit Code

| Code | Arti |
|------|------|
| `0` | Semua payload OK |
| `1` | Ada drift atau error |

## Contoh

```bat
npx restforge payload validate --config=db.env
npx restforge payload validate --config=db.env --table=users
```

## Hubungan dengan Command Lain

`validate` adalah **verdict mode**. Untuk inspect detail perbedaan kolom,
gunakan [`payload diff`](./diff.md). Untuk resolve drift secara otomatis,
gunakan [`payload sync`](./sync.md).

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
