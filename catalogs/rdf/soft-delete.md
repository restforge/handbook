# Field Soft-Delete (`softDelete`)

> Blok behavior soft-delete di RDF: visibility, reusable, field metadata FK, dan action `restore`.

## Ikhtisar (Overview)

Untuk tabel yang mengaktifkan soft-delete di SDF, `payload generate` menurunkan blok `softDelete` ke RDF beserta sejumlah field metadata FK. Endpoint yang di-generate dari RDF semacam ini berubah perilakunya: `/delete` menjadi UPDATE flag (bukan DELETE fisik), endpoint read ter-filter sesuai `visibility`, dan action `restore` tersedia untuk membalik penghapusan.

Blok ini bersifat **schema-derived**: ditulis otomatis oleh [`payload generate`](../../commands/restforge-backend/payload/generate.md) dari deklarasi [`softDelete` di SDF](../sdf/soft-delete.md), bukan dideklarasikan manual. Satu-satunya properti yang dimaksudkan untuk diubah developer per-endpoint adalah `visibility`.

```json
"softDelete": {
    "enabled": true,
    "reusable": [
        { "field": "category_code", "length": 50 }
    ],
    "visibility": "active_only"
}
```

## Bentuk Blok (Block Shape)

| Properti | Tipe | Sumber | Keterangan |
|----------|------|--------|-----------|
| `enabled` | boolean | SDF | Wajib `true` untuk mengaktifkan behavior soft-delete |
| `reusable` | array of object | SDF | Daftar kolom unique yang dimutasi suffix saat delete, entry `{ field, length }` |
| `visibility` | string | Config endpoint | Salah satu dari `active_only` (default), `deleted_only`, `include_deleted` |

Konsistensi yang ditegakkan validator untuk tiap entry `reusable`:

- `field` harus ada di `fieldName`
- `fieldValidation[field].constraints.unique` harus `true`
- `fieldValidation[field].constraints.maxLength` harus sama dengan `length` (base length, bukan physical length kolom)

`payload generate` otomatis men-set `maxLength` field reusable ke base length, sehingga input user dibatasi ke base dan mutasi suffix (`base + 38 = physical`) tidak pernah overflow.

## Semantik Visibility

| Nilai | Filter yang di-inject | Makna |
|-------|----------------------|-------|
| `active_only` | `is_deleted = FALSE` | Hanya record aktif yang terlihat (default) |
| `deleted_only` | `is_deleted = TRUE` | Hanya record terhapus yang terlihat (untuk halaman recycle-bin) |
| `include_deleted` | tanpa filter | Seluruh record terlihat |

Filter diterapkan config-driven (bukan dari request body) pada endpoint berikut:

| Endpoint | Perilaku di luar visibility |
|----------|----------------------------|
| `/list`, `/datatables` | Record tidak muncul di hasil |
| `/lookup` | Record tidak muncul di hasil |
| `/first` | `404` |
| `/update` | `404` (record terhapus tidak dapat di-update pada `active_only`) |

## Kolom Soft-Delete Tidak Masuk `fieldName`

`payload generate` mengeluarkan `is_deleted`, `deleted_at`, dan `deleted_by` dari `fieldName` RDF beserta seluruh turunannya (`datatablesQuery` SQL, `datatablesWhere`, `dateTimeFields`, `fieldValidation`). Ketiga kolom dikelola otomatis oleh runtime, sejajar dengan perlakuan kolom audit (`created_at` dkk.), sehingga tidak diekspos sebagai field API yang writable.

Validator menolak RDF yang memuat kolom soft-delete di `fieldName` tanpa blok `softDelete` ber-`enabled: true`. Guard ini menangkap RDF hasil `payload sync` atau edit manual yang kehilangan bloknya, karena tanpa blok tabel soft-delete akan diperlakukan sebagai tabel normal (`/delete` menjadi hard delete).

## Action `restore`

```json
"action": {
    "delete": true,
    "restore": true
}
```

`restore: true` menghasilkan endpoint `POST /{endpoint}/restore` yang membalik soft-delete. Validator mewajibkan `softDelete.enabled = true` bila `restore` aktif. Spesifikasi lengkap endpoint ada di [`api-spec/endpoint-restore.md`](../../api-spec/endpoint-restore.md).

## Field Metadata FK (Schema-Derived)

Soft-delete bersifat FK-aware: semantik referential integrity hard-delete ditiru di level aplikasi. Empat field top-level berikut ditulis otomatis oleh `payload generate` dan tidak dimaksudkan untuk diedit manual:

| Field | Dipersist pada | Sumber derivasi | Isi |
|-------|----------------|-----------------|-----|
| `softDeleteFkChecks` | Tabel soft-delete | Introspeksi DB | FK outbound tabel ini: `{ columns, refTable, refColumns, parentSoftDelete }` |
| `softDeleteFkParents` | Semua tabel yang ber-FK ke parent soft-delete, termasuk tabel non-soft-delete | Scan SDF | FK outbound yang parent-nya soft-delete: `{ columns, refTable, refColumns }` |
| `softDeleteFkChildren` | Tabel soft-delete yang punya anak | Scan SDF | FK inbound dari tabel anak: `{ childTable, childColumns, parentColumn, onDelete, childSoftDelete }` |
| `softDeleteCascadeTree` | Tabel soft-delete yang punya anak cascade soft-delete | Scan SDF | Closure transitif cascade: `{ master, order, nodes }` untuk walk delete rekursif |

Contoh:

```json
"softDeleteFkChildren": [
    {
        "childTable": "public.smkfk_visitor",
        "childColumns": ["category_id"],
        "parentColumn": "category_id",
        "onDelete": "restrict",
        "childSoftDelete": false
    }
]
```

### Semantik FK-Aware di Runtime

| Situasi | Hasil |
|---------|-------|
| `/create` atau `/update` anak dengan FK menunjuk parent yang sudah soft-delete | `422` |
| `/delete` master yang masih direferensikan anak `onDelete: restrict` | `409` |
| `/delete` master dengan anak `onDelete: cascade` ber-soft-delete | Master dan seluruh keturunan cascade di-soft-delete dalam satu transaction atomic |
| `/restore` record yang parent-nya masih soft-delete | `422` |
| `/restore` ketika nilai reusable sudah dipakai record aktif lain | `409` |

### Gate Validasi FK-Aware

Validator menolak kombinasi metadata yang tidak didukung:

| Gate | Kondisi yang ditolak |
|------|---------------------|
| G1 | Entry `onDelete: cascade` ke anak yang bukan soft-delete (`childSoftDelete: false`) |
| G2 | Entry `onDelete: setNull` (belum didukung) |
| G3 | Node cascade tree dengan composite primary key |
| G4, G5 | Shape `softDeleteFkChildren`/`softDeleteCascadeTree`/`softDeleteFkParents` yang malformed |

## Perubahan Perilaku Endpoint

| Endpoint | Tabel non-soft-delete | Tabel soft-delete |
|----------|----------------------|-------------------|
| `/delete` | DELETE fisik | UPDATE `is_deleted = TRUE`, `deleted_at = NOW()`, `deleted_by = <user>`, plus mutasi suffix kolom reusable. Detail di [`api-spec/endpoint-delete.md`](../../api-spec/endpoint-delete.md#perilaku-pada-tabel-soft-delete) |
| `/restore` | Tidak tersedia | Membalik soft-delete. Detail di [`api-spec/endpoint-restore.md`](../../api-spec/endpoint-restore.md) |
| `/list`, `/datatables`, `/lookup`, `/first`, `/update` | Normal | Ter-filter sesuai `visibility` |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
