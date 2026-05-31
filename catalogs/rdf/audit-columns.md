# Field Audit Columns (`auditColumns`)

Mengontrol behavior auto-injection kolom audit (`created_at`, `created_by`, `updated_at`, `updated_by`) pada operasi `create` dan `update`. Secara default RESTForge mengasumsikan tabel memiliki empat kolom audit standar tersebut dan otomatis meng-inject value-nya ke INSERT/UPDATE statement. Field `auditColumns` digunakan untuk override behavior tersebut.

## Tiga Mode Konfigurasi

| Nilai | Perilaku Generator | Use Case |
|-------|-------------------|----------|
| Tidak di-set (default) | Generator pakai default BaseModel: inject `created_at`, `created_by`, `updated_at`, `updated_by` saat INSERT/UPDATE | Tabel mengikuti konvensi audit standar RESTForge |
| `false` atau `null` | Generator emit `this.auditColumns = null` di model, runtime skip injection helper sepenuhnya | Tabel tanpa kolom audit (misal tabel log read-only, tabel master sederhana) |
| Object `{ createdAt, createdBy, updatedAt, updatedBy }` | Generator emit mapping kustom di model, runtime pakai nama kolom yang di-override | Tabel legacy dengan nama kolom audit berbeda (misal `tanggal_buat`, `dibuat_oleh`) |

## Mode `false` atau `null` (Disable Injection)

Digunakan saat tabel target **tidak** memiliki kolom audit standar. Tanpa flag ini, runtime akan mencoba inject kolom audit ke INSERT/UPDATE dan menyebabkan error 500 karena kolom tidak exist.

```json
{
    "tableName": "visitors",
    "primaryKey": "id",
    "auditColumns": false,
    "fieldName": ["id", "name", "email", "phone"],
    "action": {
        "create": true,
        "update": true,
        "datatables": true
    }
}
```

Generator akan menghasilkan model dengan baris berikut di constructor:

```javascript
// Tabel tanpa audit columns: disable injection helper
this.auditColumns = null;
```

## Mode Object (Custom Mapping)

Digunakan untuk tabel legacy yang memakai nama kolom audit non-standar.

```json
{
    "tableName": "tbl_pelanggan",
    "primaryKey": "id_pelanggan",
    "auditColumns": {
        "createdAt": "tanggal_buat",
        "createdBy": "dibuat_oleh",
        "updatedAt": "tanggal_update",
        "updatedBy": "diubah_oleh"
    },
    "fieldName": [
        "id_pelanggan", "nama_pelanggan",
        "tanggal_buat", "dibuat_oleh",
        "tanggal_update", "diubah_oleh"
    ]
}
```

Generator akan meng-inject ke INSERT/UPDATE memakai nama kolom override (`tanggal_buat`, `dibuat_oleh`, dst), bukan default RESTForge.

| Key Object | Default | Kolom Database |
|-----------|---------|----------------|
| `createdAt` | `created_at` | Timestamp insert |
| `createdBy` | `created_by` | User ID insert |
| `updatedAt` | `updated_at` | Timestamp update |
| `updatedBy` | `updated_by` | User ID update |

## Auto-Emission saat Generate Payload

Saat perintah `restforge payload generate` membaca skema database, generator otomatis mendeteksi keberadaan kolom audit standar:

| Kondisi Kolom Tabel | Output Payload |
|---------------------|----------------|
| Mengandung minimal satu dari `created_at`/`created_by`/`updated_at`/`updated_by` | `auditColumns` tidak di-emit (pakai default) |
| Tidak mengandung satupun dari kolom audit standar | `auditColumns: false` otomatis ditambahkan ke payload |

Behavior auto-emission ini memastikan payload yang di-generate dari tabel non-standar langsung kompatibel dengan runtime tanpa intervensi manual.

## Aturan Validasi

| Aturan | Pesan Error |
|--------|-------------|
| `auditColumns` harus `false`, `null`, atau object | `Property 'auditColumns' in <file> must be false, null, or object` |
| Tidak boleh array | (Sama dengan di atas, array di-treat sebagai value invalid) |
| Object value harus mengikuti shape `{ createdAt, createdBy, updatedAt, updatedBy }` | Validasi struktur dilakukan generator template saat emit `this.auditColumns` |

## Hubungan dengan `fieldValidation`

Field audit tidak perlu didaftarkan di [field-validation.md](./field-validation.md) karena value-nya di-inject oleh runtime, bukan dari request body. Bila ingin meng-expose kolom audit di response (misal di `/datatables` atau `/first`), tetap daftarkan nama kolom di `fieldName`.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
