# Field Component Engine (`components`)

Konfigurasi event lifecycle hooks pada operasi CRUD. Setiap hook merujuk ke method di file handler eksternal.

```json
{
    "components": [
        {
            "properties": {
                "filename": "components/supplier-hooks.js",
                "methods": [
                    {
                        "name": "validateSupplierCode",
                        "events": "onBeforeInsert",
                        "params": [
                            { "value": "{requestData}" },
                            { "value": "{user_id}" }
                        ]
                    },
                    {
                        "name": "notifySlack",
                        "events": "onAfterCommitInsert"
                    }
                ]
            }
        }
    ]
}
```

## Properti `components[].properties`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `filename` | string | Ya | Path file handler relatif dari root project |
| `methods` | array | Ya | Daftar method handler beserta event yang memicu |

## Properti `methods[]`

| Property | Tipe | Wajib | Keterangan |
|----------|------|-------|-----------|
| `name` | string | Ya | Nama function yang diekspor dari file handler |
| `events` | enum | Ya | Event hook (lihat tabel di bawah) |
| `params` | array | Tidak | Daftar template variable yang diteruskan ke handler |
| `transaction` | enum | Tidak | `shared` (default) atau `independent`. Lihat [Transaksi handler](#transaksi-handler) |

## Event Hook yang Didukung

| Event di dalam transaksi | Event setelah COMMIT | Trigger |
|--------------------------|----------------------|---------|
| `onBeforeInsert`, `onAfterInsert` | `onAfterCommitInsert` | Endpoint `/create` |
| `onBeforeUpdate`, `onAfterUpdate` | `onAfterCommitUpdate` | Endpoint `/update` |
| `onBeforeDelete`, `onAfterDelete` | `onAfterCommitDelete` | Endpoint `/delete` |
| `onBeforeAdjust`, `onAfterAdjust` | `onAfterCommitAdjust` | Endpoint `/adjust` |
| `onBeforeWorkflow`, `onAfterWorkflow` | `onAfterCommitWorkflow` | Endpoint `/change-status` |
| `onBeforeCompositeInsert`, `onAfterCompositeInsert` | `onAfterCommitCompositeInsert` | Endpoint `/create-composite` |
| `onBeforeCompositeUpdate`, `onAfterCompositeUpdate` | `onAfterCommitCompositeUpdate` | Endpoint `/update-composite` |

Event `onBefore*` dan `onAfter*` bersifat blocking. Handler yang mengembalikan `{ success: false }` atau melempar exception membatalkan seluruh operasi. Event `onAfterCommit*` hanya berjalan setelah data tersimpan dan tidak blocking, sehingga kegagalannya dicatat di log server tanpa mengubah response.

## Transaksi Handler

Di event `onBefore*` dan `onAfter*`, query yang dijalankan handler lewat `services.db` masuk ke transaksi operasi utama. Data yang ditulis handler ikut dibatalkan bila operasi gagal, dan handler bisa mengubah baris yang sedang diproses operasi utama tanpa menunggu lock.

| Nilai `transaction` | `services.db` di `onBefore*` dan `onAfter*` |
|---------------------|---------------------------------------------|
| `shared` (default) | Ikut transaksi operasi utama |
| `independent` | Koneksi terpisah, sehingga data yang ditulis tetap tersimpan walau operasi utama gagal |

Handler juga selalu menerima `services.dbIndependent`, yaitu koneksi terpisah yang bisa dipakai tanpa mengubah `transaction`. Di event `onAfterCommit*`, `services.db` sudah memakai koneksi terpisah karena transaksi utama telah selesai. SQLite hanya memakai satu koneksi, sehingga `independent` dan `services.dbIndependent` di SQLite tetap ikut dibatalkan bersama operasi utama.

## Nilai Field dari Event `onBefore*`

Handler `onBeforeInsert`, `onBeforeUpdate`, `onBeforeCompositeInsert`, dan `onBeforeCompositeUpdate` bisa menentukan nilai field yang disimpan dengan mengembalikan `{ success: true, data: { <field>: <nilai> } }`. Pada event composite, `data` hanya berlaku untuk field header. INSERT atau UPDATE utama disusun setelah semua handler `onBefore*` selesai.

| Aturan | Perilaku |
|--------|----------|
| Field yang tidak disebut di `data` | Tetap memakai nilai dari client |
| Beberapa handler | Isi `data` digabung sesuai urutan, dan handler berikutnya melihat hasilnya di `{requestData}` |
| Field yang bukan kolom tabel atau kolom audit | Diabaikan dan dicatat sebagai peringatan di log server |
| Primary key | Boleh diisi saat insert, perubahan saat update ditolak dengan HTTP 400 |
| Validasi | Nilai dari handler divalidasi dengan `fieldValidation` yang sama, pelanggaran dibalas HTTP 400 `VALIDATION_FAILED` |
| Mengubah `request_data` secara langsung | Tidak ikut tersimpan |

Field wajib yang selalu diisi handler ditandai dengan constraint [`filledByHook`](field-validation.md#field-yang-diisi-handler-filledbyhook).

## Template Variable di `params[].value`

| Variable | Isi |
|----------|-----|
| `{tableName}` | Nama tabel resource |
| `{requestData}` | Body request lengkap |
| `{oldData}` | Data sebelum operasi (`update`, `delete`) |
| `{newData}` | Data setelah operasi (`update`, `create`) |
| `{operation}` | Nama operasi (`insert`/`update`/`delete`) |
| `{user_id}` | User ID dari context request |
| `{timestamp}` | Timestamp eksekusi |
| `{record_id}` | Primary key record yang dioperasikan |

Detail lengkap (lifecycle flow, error handling, integrasi dengan distributed lock) dibahas di dokumentasi fitur [component engine](../../features/component-engine/README.md).

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
