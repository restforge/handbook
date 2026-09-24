# Field Policy (`fieldPolicy`)

Strategi per-kolom untuk operasi tulis (`update`, `adjust`): row locking (`lock`) dan audit trail (`audit`). Menggantikan `fieldProtection` versi lama; payload yang masih memakai `fieldProtection` ditolak generator dengan pesan yang menyebut penggantinya.

## Sintaks

```json
{
    "tableName": "product",
    "fieldName": ["product_id", "sku", "name", "qty", "unit_price", "created_at", "created_by", "updated_at", "updated_by"],
    "fieldPolicy": {
        "qty": { "strategies": ["lock", "audit"] },
        "unit_price": { "strategies": ["audit"] }
    }
}
```

Bentuk wildcard, audit seluruh kolom tulis tanpa menyebut nama satu per satu:

```json
{
    "fieldPolicy": {
        "*": { "strategies": ["audit"] }
    }
}
```

## Properti

| Properti | Tipe | Default | Wajib | Deskripsi |
|----------|------|---------|:-----:|-----------|
| `fieldPolicy.<column>.strategies` | array of string | — | ✓ | Strategi yang berlaku untuk kolom itu. Nilai valid saat ini: `"lock"`, `"audit"`, boleh keduanya sekaligus |
| `fieldPolicy.*.strategies` | array of string | — | ✓ | Bentuk wildcard, hanya menerima `["audit"]` |

### Strategi `lock`

Membungkus UPDATE dengan `SELECT ... FOR UPDATE` dalam satu transaction sebelum menulis kolom itu, mencegah race condition antara dua request yang mengubah baris yang sama secara bersamaan. Kolom mana pun di `fieldName` boleh memakai strategi ini; tidak terbatas pada kolom numerik.

### Strategi `audit`

Menulis satu baris ke tabel `<tableName>_audit` setiap kali kolom itu benar-benar berubah, berisi kolom `changed_fields` (JSON) berbentuk `{ "<kolom>": { "old": "<nilai lama>", "new": "<nilai baru>" } }`. Hanya kolom yang nilainya benar-benar berubah yang masuk `changed_fields`; kolom yang dikirim di request tapi nilainya sama dengan yang sudah tersimpan tidak dicatat. Bila tidak ada satu pun kolom ber-audit yang berubah, tidak ada baris audit yang ditulis sama sekali.

Operasi `adjust` (increment/decrement atomik, lihat [`adjust-config.md`](./adjust-config.md)) menyertakan besaran perubahannya lewat key `delta`: `{ "qty": { "operation": "adjust", "delta": "+5", "old": "10", "new": "15" } }`.

Strategi `audit` dapat dipakai sendiri tanpa `lock`. Kolom yang hanya memakai `audit` tidak melalui `SELECT ... FOR UPDATE`; pembacaan baris sebelum-ubah cukup lewat `SELECT` biasa dalam transaction yang sama.

## Bentuk Wildcard (`"*"`)

Satu entri `"*"` berarti "audit seluruh kolom tulis", diperluas oleh generator menjadi entri per kolom sebelum payload divalidasi lebih lanjut:

- Hanya menerima strategi `"audit"`. Wildcard tidak dapat memakai `"lock"` karena mengunci seluruh kolom sekaligus tidak dibutuhkan pola standar; kolom yang memang perlu `lock` dideklarasikan satu per satu.
- Himpunan kolom yang di-audit = `schemaFields` (bila payload mendeklarasikan array itu) atau `fieldName`, dikurangi primary key, kolom audit (`created_at`/`created_by`/`updated_at`/`updated_by` efektif), dan `concurrency.versionColumn` (bila blok [`concurrency`](./concurrency.md) ada).
- Entri kolom eksplisit yang sudah ada di `fieldPolicy` tetap dipertahankan apa adanya, tidak ditimpa oleh hasil ekspansi wildcard.
- Kunci `"*"` sendiri dihapus setelah ekspansi; payload yang tersimpan di server hanya berisi entri per kolom.

## Aturan Validasi

| Aturan | Pesan Error |
|--------|-------------|
| `fieldProtection` (nama lama) tidak lagi didukung | `'fieldProtection' in <file> has been replaced by 'fieldPolicy'. Example: { "fieldPolicy": { "<column>": { "strategies": ["lock", "audit"] } } }` |
| `fieldPolicy` harus object | `fieldPolicy in <file> must be a non-null object` |
| Nama kolom harus ada di `fieldName` | `fieldPolicy column '<column>' in <file> is not in fieldName` |
| Konfigurasi per kolom harus object | `fieldPolicy.<column> in <file> must be a non-null object` |
| `strategies` wajib array string tidak kosong | `fieldPolicy.<column>.strategies in <file> is required and must be a non-empty array of strings` |
| Strategi yang belum tersedia (`condition`, `version`) | `fieldPolicy.<column>.strategies includes '<strategy>' which is planned for Phase 2 of fieldPolicy roadmap. Currently only 'lock' and 'audit' are supported.` |
| Strategi tidak dikenal | `fieldPolicy.<column>.strategies includes '<strategy>' which is not valid. Valid strategies in Phase 1: lock, audit` |
| Key lain selain `strategies` per kolom | `fieldPolicy.<column>.<key> is not supported in Phase 1 of fieldPolicy roadmap` |
| Wildcard `"*"` harus object | `fieldPolicy.* in <file> must be a non-null object` |
| Wildcard hanya menerima key `strategies` | `fieldPolicy.* in <file> only supports the 'strategies' key (found '<key>')` |
| `strategies` wildcard wajib array string tidak kosong | `fieldPolicy.*.strategies in <file> is required and must be a non-empty array of strings` |
| Wildcard hanya menerima strategi `audit` | `fieldPolicy.* in <file> only supports strategy 'audit' (found '<strategy>'). Wildcard cannot be used to lock every column; declare 'lock' per column instead` |
| Ekspansi wildcard tidak menyisakan kolom sama sekali | `fieldPolicy.* in <file> expanded to zero columns (all candidate columns are excluded as the primary key, audit columns, or concurrency.versionColumn)` |

## Dukungan Dialect

| Dialect | Strategi `lock` | Strategi `audit` |
|---------|:----------------:|:------------------:|
| PostgreSQL | ✓ | ✓ |
| MySQL | ✓ | ✓ |
| Oracle | ✓ | ✓ |
| SQLite | ✗ | ✗ |

SQLite belum mendukung `fieldPolicy` sama sekali; deklarasi ini diabaikan oleh template SQLite. Karena migrasi audit table baru dijalankan setelah seluruh file endpoint ditulis, `endpoint create --database=sqlite` untuk payload ber-`fieldPolicy` strategi `audit` tetap menulis seluruh file generate lebih dulu sebelum gagal di tahap migrasi. Untuk SQLite, hindari strategi `audit`.

## Migrasi Audit Table

Payload dengan strategi `audit` (per kolom maupun lewat wildcard) membuat `endpoint create` menulis DDL audit table ke `migrations/audit/<tableName>_audit.sql` dan, secara default, mengeksekusinya ke database target. Flag `--no-audit-migration` menahan eksekusi (file SQL tetap ditulis) untuk pipeline yang ingin review DDL dulu. Detail flag ini ada di [`../../commands/restforge-backend/endpoint/create.md`](../../commands/restforge-backend/endpoint/create.md).

Skema tabel audit bersifat generic dan sama untuk seluruh tabel sumber, tidak berubah mengikuti isi `fieldPolicy`:

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `audit_id` | string (UUID) | Primary key baris audit |
| `record_pk` | text | Nilai primary key baris sumber yang diaudit |
| `operation` | string | `"update"` atau `"adjust"` |
| `changed_fields` | JSON | `{ "<kolom>": { "old", "new" } }`, ditambah `operation`/`delta` untuk `adjust` |
| `metadata` | JSON | Konteks tambahan (nullable) |
| `changed_by` | string | User ID yang mengeksekusi operasi (nullable) |
| `changed_at` | timestamp | Waktu baris audit ditulis |

## Interaksi dengan `concurrency`

Kolom `versionColumn` (lihat [`concurrency.md`](./concurrency.md)) selalu dikecualikan dari ekspansi wildcard `"*"` dan dari perbandingan `noChanges`, sehingga kenaikan versinya sendiri tidak pernah tercatat sebagai perubahan kolom biasa di `changed_fields`. Kedua fitur berjalan independen: `concurrency` menentukan apakah UPDATE boleh berjalan (syarat versi), `fieldPolicy` menentukan bagaimana UPDATE itu dieksekusi (locking) dan apa yang dicatat setelahnya (audit).

Saat `concurrency` dideklarasikan, request yang tidak mengubah nilai kolom apa pun (dibandingkan dengan baris saat ini di database) tidak menjalankan UPDATE, tidak menaikkan versi, dan tidak menulis baris audit; server membalas `noChanges: true`. Detail bentuk respons ada di [`../../api-spec/endpoint-update.md`](../../api-spec/endpoint-update.md#response-sukses). Tanpa blok `concurrency`, request semacam ini tetap menjalankan UPDATE seperti sebelumnya (perbandingan tanpa-perubahan hanya aktif bila ada versi baris yang dijaga).

---

**Lihat juga**: [`concurrency.md`](./concurrency.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
