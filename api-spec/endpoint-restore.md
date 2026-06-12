# Restore — Endpoint Pemulihan Data Soft-Delete

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Metode HTTP** | `POST` |
| **URL** | `/api/{project}/{endpoint}/restore` |
| **Content-Type** | `application/json` |
| **HTTP Status Sukses** | `200 OK` |
| **Database** | PostgreSQL (soft-delete Fase 1) |
| **Prasyarat RDF** | `action.restore = true` dan `softDelete.enabled = true` |
| **Parameter Wajib** | `where` (menentukan record terhapus yang akan dipulihkan) |
| **Cache** | Invalidasi otomatis setelah restore berhasil |

---

## Ikhtisar

Endpoint `/restore` membalik operasi soft-delete: record yang sebelumnya ditandai terhapus dikembalikan menjadi aktif. Endpoint ini hanya tersedia pada resource dengan blok [`softDelete`](../catalogs/rdf/soft-delete.md) aktif dan `action.restore = true` di RDF.

Operasi berjalan atomic dalam satu transaction: record terhapus dikunci (`SELECT FOR UPDATE`), suffix mutasi pada kolom reusable di-strip kembali ke nilai asli, konsistensi diperiksa, lalu ketiga kolom soft-delete di-reset:

```sql
is_deleted = FALSE, deleted_at = NULL, deleted_by = NULL
```

Kolom audit `updated_*` tidak disentuh. Record lama tanpa suffix (data legacy) tetap dapat di-restore, langkah strip dilewati.

**Contoh URL:**
```
POST http://localhost:3000/api/mini-inventory/category/restore
```

---

## Format Request

### Parameter

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|:-----:|------------|
| `where` | array / object | **Ya** | Kondisi untuk menentukan record terhapus yang akan dipulihkan. Lihat [Format WHERE](README.md#format-where-clause) |

### Contoh Request

```json
{
  "where": [
    { "key": "category_id", "value": "550e8400-e29b-41d4-a716-446655440000" }
  ]
}
```

---

## Format Response

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "Successfully restored 1 record(s)",
  "restored_count": 1,
  "restored_items": [
    {
      "category_id": "550e8400-e29b-41d4-a716-446655440000",
      "category_code": "CAT-001",
      "category_name": "Beverages",
      "is_deleted": false,
      "deleted_at": null,
      "deleted_by": null
    }
  ],
  "timestamp": "2026-06-12T15:00:00.000Z"
}
```

| Field | Tipe | Keterangan |
|-------|------|------------|
| `success` | boolean | `true` jika operasi berhasil |
| `message` | string | `"Successfully restored {n} record(s)"` |
| `restored_count` | number | Jumlah record yang berhasil dipulihkan |
| `restored_items` | array | Data record setelah dipulihkan (kolom reusable sudah kembali ke nilai asli) |
| `timestamp` | string | Waktu eksekusi (ISO 8601) |

### Response Error

#### 400 — Payload Kosong / WHERE Tidak Ada / Format WHERE Tidak Valid

Sama dengan endpoint [`/delete`](endpoint-delete.md#response-error): payload kosong, `where` absen, atau struktur `where` tidak dikenali ditolak dengan 400 beserta contoh format.

#### 404 — Data Tidak Ditemukan atau Tidak Terhapus

Terjadi ketika tidak ada record yang cocok dengan kondisi WHERE, atau record yang dituju masih aktif (tidak dalam keadaan terhapus).

```json
{
  "success": false,
  "error": "Data not found",
  "message": "category data not found or not deleted",
  "timestamp": "2026-06-12T15:00:00.000Z"
}
```

#### 409 — Nilai Reusable Sudah Dipakai Record Aktif Lain

Terjadi ketika nilai asli kolom reusable (hasil strip suffix) sudah dipakai oleh record aktif lain sehingga restore akan melanggar UNIQUE.

```json
{
  "success": false,
  "error": "Conflict",
  "message": "Cannot restore: 'category_code' value 'CAT-001' is already in use by an active record",
  "timestamp": "2026-06-12T15:00:00.000Z"
}
```

Race condition saat UPDATE (UNIQUE violation `23505`) menghasilkan 409 dengan pesan `"Cannot restore: the reusable code is already in use by an active record (unique constraint)"`.

#### 422 — Parent yang Dirujuk Tidak Aktif

Terjadi ketika record yang dipulihkan punya FK ke parent yang hilang atau masih dalam keadaan soft-delete. Record anak tidak boleh aktif sementara parent-nya terhapus.

```json
{
  "success": false,
  "error": "Unprocessable Entity",
  "message": "Cannot restore: referenced parent record in 'public.category_group' is missing or soft-deleted",
  "timestamp": "2026-06-12T15:00:00.000Z"
}
```

FK bernilai NULL dilewati dari pemeriksaan (FK nullable tidak mewajibkan parent).

#### 500 — Internal Server Error

```json
{
  "success": false,
  "error": "Internal Server Error",
  "message": "An error occurred while restoring category data",
  "details": "detail error teknis (hanya di development mode)",
  "timestamp": "2026-06-12T15:00:00.000Z"
}
```

---

## Perilaku Cache

| Aspek | Nilai |
|-------|-------|
| Di-cache | Tidak (write operation) |
| Invalidasi | Ya, setelah restore berhasil |
| Key pattern | `rf:{project}:{endpoint}:*` |

---

## Urutan Operasi (Operation Flow)

```
Request masuk
  → Validasi parameter WHERE
  → BEGIN transaction
  → SELECT FOR UPDATE record terhapus (is_deleted = TRUE)
  → Strip suffix kolom reusable → cek konflik nilai asli (409)
  → Cek parent FK aktif (422)
  → UPDATE reset is_deleted/deleted_at/deleted_by
  → COMMIT
  → Invalidasi cache
  → Response 200
```

---

## Fitur Terkait

| Fitur | Dokumen | Relevansi |
|-------|---------|-----------|
| Soft-Delete (RDF) | [`catalogs/rdf/soft-delete.md`](../catalogs/rdf/soft-delete.md) | Blok `softDelete`, visibility, metadata FK |
| Soft-Delete (SDF) | [`catalogs/sdf/soft-delete.md`](../catalogs/sdf/soft-delete.md) | Deklarasi schema, kolom kontrak, reusable |
| Delete | [`endpoint-delete.md`](endpoint-delete.md) | Operasi kebalikan: soft-delete pada tabel ber-`softDelete` |
