# RDF Examples

> File `.json` Resource Definition File untuk skenario umum. Copy ke folder `payload/` di project, sesuaikan nama tabel atau kolom, lalu jalankan `npx restforge endpoint create`.

## Daftar Contoh

| File | Skenario | Fokus |
|------|----------|-------|
| [category.json](./category.json) | Master data sederhana | CRUD dasar, primary key UUID, sanitization (`trim`, `uppercase`) |
| [supplier.json](./supplier.json) | Master data lengkap | `fieldValidation` comprehensive: preset `format` (email/phone/url), `pattern`, `enum`, `decimal` dengan `precision`/`scale` |
| [item-product.json](./item-product.json) | Master data dengan custom query | `datatablesQuery` dengan `file:` reference, `fieldNameLookup` dengan SQL concat di `text` untuk display gabungan |

## Konvensi

- Setiap file memakai struktur baru dengan `fieldValidation`. Struktur lama (hanya `fieldName`) tetap didukung untuk backward compatibility. Detail di [catalogs/rdf/field-validation.md](../../catalogs/rdf/field-validation.md).
- Primary key memakai UUID v7 dengan `autoGenerate: true` (lihat [field-validation.md](../../catalogs/rdf/field-validation.md) section Behavior `autoGenerate`).
- Kolom audit (`created_at`, `created_by`, `updated_at`, `updated_by`) tidak perlu didaftarkan di `fieldValidation` karena di-handle otomatis oleh runtime. Detail di [catalogs/rdf/audit-columns.md](../../catalogs/rdf/audit-columns.md).
- Urutan entry di `fieldName` menentukan prioritas tampilan default (datatables, form).

## Cara Pakai

1. Copy file `.json` ke folder `<project>/payload/`
2. Sesuaikan `tableName`, kolom di `fieldName`, dan aturan di `fieldValidation` agar match dengan tabel database
3. Untuk contoh dengan `datatablesQuery: "file:..."`, buat juga file SQL di lokasi yang dirujuk (konvensi: `payload/query/<nama>.sql`)
4. Generate endpoint dengan command:

   ```bash
   npx restforge endpoint create --resource=<nama-file-tanpa-ekstensi>
   ```

## Referensi

- [Resource Definition File (RDF)](../../catalogs/rdf/) — Spesifikasi lengkap RDF
- [Field Validasi](../../catalogs/rdf/field-validation.md) — Detail constraint per tipe data
- [Field Lookup](../../catalogs/rdf/field-lookup.md) — Konfigurasi endpoint `/lookup`
- [Audit Columns](../../catalogs/rdf/audit-columns.md) — Kolom audit auto-managed
- [Endpoint Create Command](../../commands/restforge-backend/endpoint/create.md) — CLI untuk generate endpoint

---

**Lihat juga**: [`examples/`](../) · [`README`](../../README.md)
