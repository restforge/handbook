# Definisi Soft-Delete (`softDelete`)

> Deklarasi soft-delete di level schema: tiga kolom kontrak, CHECK konsistensi, partial index, dan mekanisme reuse kolom unique.

## Ikhtisar (Overview)

Blok `softDelete` mengaktifkan fitur soft-delete untuk sebuah tabel. Record yang dihapus tidak benar-benar di-DELETE dari database, melainkan ditandai lewat tiga kolom kontrak (`is_deleted`, `deleted_at`, `deleted_by`). SDF adalah sumber deklarasi tunggal: blok ini yang menentukan emisi DDL (kolom, CHECK constraint, partial index) sekaligus menjadi sumber derivasi blok `softDelete` di RDF saat [`payload generate`](../../commands/restforge-backend/payload/generate.md).

> **Dukungan dialect (Fase 1):** soft-delete hanya tersedia untuk PostgreSQL. Pada dialect lain, deklarasi `softDelete` belum didukung dan kolom bernama soft-delete tidak mendapat perlakuan khusus saat introspect.

```javascript
module.exports = ({ defineModel }) => defineModel('visitor_categories', {
  fields: {
    category_id:   'string:36 pk',
    category_code: 'string:88 notnull unique',   // base 50 + suffix 38 = 88
    category_name: 'string:100 notnull',
    is_deleted:    'boolean notnull default:false',
    deleted_at:    'timestamp',
    deleted_by:    'string:70',
  },
  softDelete: {
    enabled: true,
    reusable: [
      { field: 'category_code', length: 50 }
    ]
  }
});
```

## Bentuk Blok (Block Shape)

| Properti | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `enabled` | boolean | Ya | Mengaktifkan soft-delete untuk tabel ini |
| `reusable` | array of object | Tidak | Daftar kolom unique yang nilainya boleh dipakai ulang setelah record di-soft-delete |

Setiap entry `reusable` berbentuk `{ field, length }`:

| Properti | Tipe | Keterangan |
|----------|------|-----------|
| `field` | string | Nama kolom yang dideklarasikan di `fields` |
| `length` | integer > 0 | Base length, yaitu panjang maksimal nilai yang boleh di-input user |

Key di luar `enabled` dan `reusable` ditolak dengan error saat load schema.

## Tiga Kolom Kontrak (Column Contract)

Bila `enabled = true`, ketiga kolom berikut **wajib** dideklarasikan di `fields` dengan tipe persis seperti tabel di bawah:

| Kolom | Tipe Logical | Keterangan |
|-------|--------------|-----------|
| `is_deleted` | `boolean` | Flag terhapus. Disarankan `notnull default:false` |
| `deleted_at` | `timestamp` | Waktu penghapusan, nullable |
| `deleted_by` | `string` | Pelaku penghapusan, nullable |

Hubungan kolom dan blok bersifat dua arah (biconditional):

| Kondisi | Hasil |
|---------|-------|
| `enabled = true`, ketiga kolom lengkap dan tipe sesuai | Valid |
| `enabled = true`, ada kolom yang hilang | ERROR saat validasi (daftar kolom missing disebutkan) |
| Kolom soft-delete dideklarasikan tetapi `enabled` bukan `true` | ERROR saat validasi |
| Tipe kolom tidak sesuai kontrak | ERROR saat validasi |

Konsekuensinya, nama `is_deleted`/`deleted_at`/`deleted_by` adalah namespace yang direservasi kontrak soft-delete. Kolom dengan nama tersebut tidak dapat dipakai sebagai kolom biasa di tabel yang dikelola RESTForge.

## Kolom Reusable dan Suffix Length

Soft-delete menyisakan row di database, sehingga UNIQUE constraint tetap berlaku terhadap row yang sudah terhapus. Agar nilai unique (misal business code) dapat dipakai ulang oleh record baru, kolom yang dideklarasikan `reusable` akan dimutasi nilainya saat delete dengan format `{nilai}##{uuidv7}`.

Konsekuensi terhadap panjang kolom:

```
physical length >= base length + 38        // 38 = '##' (2) + uuidv7 (36)
```

Contoh: `length: 50` menuntut deklarasi kolom minimal `string:88`. Bila kurang, validasi gagal dengan error yang menyebutkan panjang declared dan required.

Syarat kolom `reusable`:

| Syarat | Bila dilanggar |
|--------|----------------|
| Terdeklarasi di `fields` | ERROR (unknown field) |
| Tipe `string` atau `text` | ERROR (value-mutation butuh tipe string) |
| Punya single-column UNIQUE | ERROR (tanpa UNIQUE, atau hanya composite UNIQUE) |
| `length` integer positif | ERROR |

## Gate Kelayakan UNIQUE (Unique Reusability Gate)

Saat `enabled = true`, seluruh UNIQUE di tabel diperiksa kelayakannya:

- **Composite UNIQUE** ditolak, karena tidak dapat dibebaskan lewat mutasi nilai. Pembuatan ulang row dengan key yang sama setelah soft-delete akan melanggar UNIQUE tersebut.
- **Single-column UNIQUE bertipe non-string** ditolak dengan alasan yang sama (mutasi suffix hanya berlaku untuk string/text).

Pesan error menyarankan dua jalan keluar: gunakan hard-delete untuk tabel process-driven (balance, ledger, snapshot, period-close), atau ubah UNIQUE menjadi single-column string business code.

## DDL yang Di-emit (Emitted DDL)

### CHECK Konsistensi

Setiap tabel soft-delete mendapat CHECK constraint dengan nama deterministik `chk_<table>_soft_delete_consistency`:

```sql
CONSTRAINT chk_visitor_categories_soft_delete_consistency CHECK (
    (is_deleted = TRUE  AND deleted_at IS NOT NULL AND deleted_by IS NOT NULL)
 OR (is_deleted = FALSE AND deleted_at IS NULL     AND deleted_by IS NULL)
)
```

CHECK ini menjamin ketiga kolom selalu konsisten di level data, terlepas dari jalur tulis yang dipakai.

### Partial Index

Index non-unique pada tabel soft-delete di-emit sebagai partial index di PostgreSQL:

```sql
CREATE INDEX idx_visitor_categories_category_name
    ON visitor_categories (category_name) WHERE is_deleted = FALSE
```

UNIQUE constraint **tidak** di-partial-kan di dialect mana pun. Uniqueness tetap penuh mencakup row terhapus, dan reuse ditangani lewat mutasi suffix, bukan partial unique index.

## Round-Trip Introspect

`schema introspect` mengenali tabel soft-delete yang lengkap (tiga kolom sesuai tipe + CHECK konsistensi) dan menuliskan kembali blok `softDelete: { enabled: true }` di SDF hasil introspect. Tabel dengan kolom soft-delete yang tidak lengkap atau tidak sesuai kontrak justru memblokir introspect, lihat [kondisi blokir di halaman command](../../commands/restforge-backend/schema/introspect.md#kondisi-blokir-soft-delete-strict).

## Keterkaitan dengan RDF

Blok `softDelete` di RDF **diturunkan dari SDF** saat `payload generate`, bukan ditulis manual. Tabel yang punya kolom soft-delete di database wajib dideklarasikan di SDF dengan blok valid agar `payload generate` berhasil. Detail derivasi dan aturan error ada di [`commands/restforge-backend/payload/generate.md`](../../commands/restforge-backend/payload/generate.md#derivasi-soft-delete-dan-flag---schema-path) dan kontrak sisi RDF ada di [`catalogs/rdf/soft-delete.md`](../rdf/soft-delete.md).

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
