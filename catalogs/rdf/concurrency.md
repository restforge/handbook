# Field Concurrency (`concurrency`)

Optimistic concurrency control berbasis versi baris untuk operasi `update` dan `update-composite`. Tanpa blok ini, kedua endpoint tetap berjalan seperti sekarang: request terakhir yang masuk selalu menimpa baris, tanpa pemeriksaan apa pun terhadap versi yang dibaca client sebelumnya.

## Sintaks

```json
{
    "tableName": "product",
    "primaryKey": "product_id",
    "fieldName": ["product_id", "sku", "name", "price", "row_version", "created_at", "created_by", "updated_at", "updated_by"],
    "concurrency": {
        "versionColumn": "row_version",
        "compare": "version"
    }
}
```

## Properti

| Properti | Tipe | Default | Wajib | Deskripsi |
|----------|------|---------|:-----:|-----------|
| `versionColumn` | string | — | ✓ | Nama kolom yang menyimpan versi baris. Harus ada di `fieldName` (dan di `schemaFields` bila payload mendeklarasikannya) |
| `compare` | string | `"version"` | ✗ | Cara pembandingan versi. `"version"`: kolom integer yang dinaikkan 1 setiap UPDATE yang menulis. `"timestamp"`: fallback ke kolom `updatedAt` efektif, hanya untuk PostgreSQL |

`versionColumn` pada mode `"version"` dideklarasikan di SDF sebagai kolom integer biasa (default `1`, `NOT NULL`), lalu ditambahkan ke tabel lewat `schema migrate` seperti kolom lain. Tidak ada tooling SDF khusus untuk kolom ini; generator RDF tidak membuat kolom apa pun, hanya membaca deklarasinya di `fieldName`.

## Aturan Validasi

Dijalankan oleh `validateConcurrency` sebelum ekspansi `fieldPolicy["*"]` (lihat [`field-policy.md`](./field-policy.md)), sehingga `versionColumn` yang dipakai sebagai pengecualian di sana sudah terjamin valid.

| Aturan | Pesan Error |
|--------|-------------|
| `concurrency` harus object | `concurrency in <file> must be a non-null object` |
| Key tidak dikenal | `concurrency.<key> in <file> is not supported. Valid keys: versionColumn, compare` |
| `versionColumn` wajib, string tidak kosong | `concurrency.versionColumn in <file> is required and must be a non-empty string` |
| `versionColumn` harus ada di `fieldName` | `concurrency.versionColumn '<versionColumn>' in <file> is not in fieldName` |
| `versionColumn` harus ada di `schemaFields` (bila payload mendeklarasikan array itu) | `concurrency.versionColumn '<versionColumn>' in <file> is not in schemaFields` |
| `compare` hanya `"version"` atau `"timestamp"` | `concurrency.compare in <file> must be either 'version' or 'timestamp' (found '<value>')` |
| Mode `version`: `versionColumn` tidak boleh sama dengan primary key | `concurrency.versionColumn '<versionColumn>' in <file> cannot be the same as the primary key ('<pk>')` |
| Mode `version`: `versionColumn` tidak boleh salah satu kolom audit | `concurrency.versionColumn '<versionColumn>' in <file> cannot be one of the audit columns (<daftar kolom>)` |
| Mode `timestamp`: `auditColumns` wajib aktif | `concurrency.compare 'timestamp' in <file> requires auditColumns to be enabled (versionColumn must match the effective updatedAt column)` |
| Mode `timestamp`: `versionColumn` wajib sama dengan kolom `updatedAt` efektif | `concurrency.compare 'timestamp' in <file> requires concurrency.versionColumn to equal the updatedAt column ('<updatedAtColumn>'), found '<versionColumn>'` |
| Mode `timestamp` pada dialect selain PostgreSQL, diperiksa `endpoint create` saat `--database` sudah diketahui, sebelum satu file pun ditulis | `concurrency.compare 'timestamp' in payload <file> is only supported for --database=postgres (found '<dialect>'). Use an integer versionColumn (concurrency.compare 'version', the default) for other dialects.` |

## Contoh

RDF lengkap dengan `concurrency` mode `version` dan `fieldPolicy["*"]` (lihat [`field-policy.md`](./field-policy.md) untuk penjelasan wildcard):

```json
{
    "tableName": "product",
    "primaryKey": "product_id",
    "fieldName": [
        "product_id", "sku", "name", "price", "active", "spec",
        "row_version",
        "created_at", "created_by", "updated_at", "updated_by"
    ],
    "concurrency": {
        "versionColumn": "row_version",
        "compare": "version"
    },
    "fieldPolicy": {
        "*": { "strategies": ["audit"] }
    },
    "action": {
        "create": true,
        "read": true,
        "first": true,
        "update": true,
        "delete": true,
        "datatables": true
    }
}
```

Mode `timestamp` (khusus PostgreSQL, `updated_at` sebagai token):

```json
{
    "tableName": "product",
    "primaryKey": "product_id",
    "fieldName": ["product_id", "sku", "name", "created_at", "created_by", "updated_at", "updated_by"],
    "concurrency": {
        "versionColumn": "updated_at",
        "compare": "timestamp"
    }
}
```

## Perilaku Runtime

Pemeriksaan versi terjadi **di dalam** statement `UPDATE` lewat klausa `WHERE ... AND <versionColumn> = ?`, bukan lewat `SELECT` terpisah yang dibandingkan di application code. Berlaku pada `/update` dan pada baris **master** di `/update-composite` (detail tidak berversi). Kontrak lengkap request/response ada di [`../../api-spec/endpoint-update.md`](../../api-spec/endpoint-update.md#pemeriksaan-versi-baris-optimistic-concurrency) dan [`../../api-spec/endpoint-update-composite.md`](../../api-spec/endpoint-update-composite.md).

Ringkasan:

| Kondisi | Hasil |
|---------|-------|
| Client mengirim `options.expectedVersion` yang cocok | UPDATE berjalan, versi naik 1 (mode `version`) |
| Client mengirim token yang tidak cocok, baris masih ada | `409 Conflict` dengan `details.currentVersion` |
| Baris tidak ditemukan sama sekali | `404` seperti biasa |
| Client tidak mengirim `options.expectedVersion` | UPDATE tetap berjalan tanpa syarat versi di `WHERE`, plus warning sekali di log server per instance model |
| Token bukan integer (mode `version`) atau string kosong (mode `timestamp`) | `400` |

**Versi tetap naik meski client tidak mengirim token.** Selama `concurrency.versionColumn` dideklarasikan, setiap UPDATE yang benar-benar menulis (bukan `noChanges`, lihat [`field-policy.md`](./field-policy.md#interaksi-dengan-concurrency)) menaikkan `versionColumn` satu, terlepas apakah request itu membawa token atau tidak. Hanya syarat `WHERE ... AND versionColumn = ?` yang bersyarat token; kenaikan versinya sendiri tidak. Ini mencegah lost update pada campuran client lama (tanpa token) dan client baru (ber-token): tulisan client lama tetap terlihat oleh client ber-token berikutnya sebagai perubahan versi, bukan diam-diam terlewati.

RDF tanpa blok `concurrency` menghasilkan output generate yang identik dengan sebelum fitur ini ada; tidak ada `SELECT` tambahan, tidak ada syarat versi, tidak ada perubahan pada endpoint mana pun.

## Tabel Dukungan Dialect

| `compare` | PostgreSQL | MySQL | Oracle | SQLite |
|-----------|:----------:|:-----:|:------:|:------:|
| `"version"` (kolom integer) | ✓ | ✓ | ✓ | ✓ |
| `"timestamp"` (kolom `updatedAt`) | ✓ | ✗ | ✗ | ✗ |

## Batasan

- `compare: "timestamp"` hanya berlaku untuk PostgreSQL dan hanya untuk kolom `updatedAt` efektif (default `updated_at`, atau hasil mapping `auditColumns`). Perbandingan dilakukan pada presisi milidetik, sehingga dua tulisan yang terjadi dalam milidetik yang sama tidak terbedakan sebagai versi berbeda. Kolom integer (`compare: "version"`, default) disarankan untuk kasus umum karena tidak punya batasan presisi seperti ini.
- Token `versionColumn` dari response `first` dikirim apa adanya sebagai `options.expectedVersion` pada request update berikutnya. Token itu berpola `DATETIMEFORMAT`, dan runtime mengembalikannya ke bentuk kanonik sebelum membandingkan, sehingga pola aktif tidak memengaruhi hasil. `DATETIMEFORMAT` wajib memuat `.SSS`: tanpa milidetik, token kehilangan presisi dan setiap update ditolak sebagai konflik (lihat [`datetime-fields.md`](./datetime-fields.md#milidetik)).
- Lock distribusi (per-record WRITE lock) tidak digantikan oleh `concurrency`. Keduanya saling melengkapi: lock distribusi mencegah dua request memproses record yang sama secara bersamaan, sedangkan `concurrency` memastikan client menyimpan perubahan berdasarkan versi data yang benar-benar terbaru saat form dibuka.

---

**Lihat juga**: [`field-policy.md`](./field-policy.md) · [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
