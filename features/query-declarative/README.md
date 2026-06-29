# Query Declarative

Query Declarative berfungsi untuk menentukan sumber data dan query SQL setiap endpoint melalui payload JSON, tanpa menulis kode model secara manual. Setiap properti query melayani endpoint tertentu dan diresolusi secara otomatis berdasarkan prioritas saat module di-generate.

Halaman ini berisi panduan tugas. Untuk spec field RDF yang lengkap, lihat referensi di [`catalogs/rdf/data-source.md`](../../catalogs/rdf/data-source.md).

> **Database:** PostgreSQL, MySQL, Oracle, SQLite. Satu payload menghasilkan behavior identik di seluruh dialect.

---

## Referensi Cepat (Quick Reference)

| Properti | Nilai |
|----------|-------|
| Properti query | `datatablesQuery`, `viewQuery`, `viewName`, `exportQuery`, `detailQuery` |
| Deklarasi | Inline SQL string atau file reference (`file:query/xxx.sql`) |
| Resolusi `/datatables` | `datatablesQuery` lalu fallback `SELECT * FROM readSource` |
| Resolusi `/read`, `/first` | `viewName` lalu `viewQuery` lalu `tableName` |
| Resolusi `/lookup` | `viewName` lalu `viewQuery` lalu `tableName` |
| Resolusi `/export` | `exportQuery` lalu fallback `SELECT {fieldName} FROM {tableName}` |
| Resolusi `/read-composite` | Header sama dengan `/read`; detail memakai `detailQuery` |
| Introspeksi spec | `npx restforge catalog query-declarative` |

---

## Daftar Properti Query (Query Properties)

Setiap properti query melayani endpoint tertentu. Tabel berikut memetakan properti ke endpoint dan menunjukkan dukungan file reference.

| Properti | Wajib | File Reference | Endpoint yang Dilayani |
|----------|-------|----------------|------------------------|
| `datatablesQuery` | Ya | Ya | `/datatables` |
| `viewQuery` | Tidak | Ya | `/read`, `/first`, `/lookup`, `/read-composite` (header) |
| `viewName` | Tidak | Tidak | `/read`, `/first`, `/lookup`, `/read-composite` (header) |
| `exportQuery` | Tidak | Ya | `/export` |
| `masterDetail.detailConfig.detailQuery` | Tidak | Ya | `/read-composite` (detail) |

`viewName` tidak mendukung file reference karena berisi nama database object, bukan konten SQL.

---

## Prioritas Resolusi per Endpoint (Resolution Priority)

Setiap endpoint menentukan query SQL berdasarkan urutan prioritas berikut. Prioritas pertama yang tersedia di payload akan dipakai.

```
/datatables      →  datatablesQuery  →  fallback: SELECT * FROM readSource
/read            →  viewName  →  viewQuery  →  tableName
/first           →  viewName  →  viewQuery  →  tableName
/lookup          →  viewName  →  viewQuery  →  tableName
/export          →  exportQuery  →  fallback: SELECT {fieldName} FROM {tableName}
/read-composite  →  Header: viewName  →  viewQuery  →  tableName
                    Detail: detailQuery  →  fallback: SELECT * FROM {detailTable}
                                           WHERE {foreignKey} = ? ORDER BY line_number
```

> **Catatan:** `datatablesQuery` tidak menjadi fallback untuk `/read`. Kedua endpoint memiliki sumber data yang independen. Endpoint `/read`, `/first`, dan `/lookup` berbagi resolusi tiga tingkat yang sama, sehingga `viewQuery` ikut berlaku pada `/lookup`. Operasi write (INSERT, UPDATE, DELETE) selalu memakai `tableName`, tidak terpengaruh konfigurasi query di atas.

---

## Read Source dan Write Source (Read vs Write Source)

Operasi read dan write memakai sumber yang terpisah. Konfigurasi query declarative hanya memengaruhi read.

| Operasi | Source | Ditentukan oleh |
|---------|--------|-----------------|
| Read (SELECT) | `readSource` | `viewName` lalu `tableName` |
| Write (INSERT, UPDATE, DELETE) | `writeSource` | Selalu `tableName` |

---

## Dokumentasi Lengkap (Full Documentation)

| Dokumen | Isi |
|---------|-----|
| [Query List dan Read](read-queries.md) | `datatablesQuery`, `viewQuery`, `viewName` untuk `/datatables`, `/read`, `/first`, `/lookup` |
| [Query Export](export-queries.md) | `exportQuery` dan format kolom untuk file Excel |
| [Master-Detail Composite](master-detail.md) | `detailQuery` untuk endpoint `/read-composite` |
| [Query sebagai File](file-reference.md) | Format `file:`, struktur folder, dan proses generation |
| [Kolom Response dan Validasi](field-mapping.md) | Peran `fieldName` dan `fieldValidation` terhadap query |
| [Introspeksi via CLI](catalog-cli.md) | Command `catalog query-declarative` untuk discovery spec |

---

## Catatan Teknis (Technical Notes)

### Konversi SQL Otomatis (Automatic SQL Conversion)

Query yang ditulis dalam sintaks PostgreSQL dikonversi secara otomatis ke sintaks dialect target saat generation.

| Dialect | Konversi |
|---------|----------|
| Oracle | `ILIKE` ke `LIKE`, `NOW()` ke `SYSDATE`, `LIMIT/OFFSET` ke `ROWNUM` |
| MySQL | `ILIKE` ke `LIKE` |

### Subquery Wrapping (Subquery Wrapping)

Apabila `datatablesQuery` atau `viewQuery` mengandung JOIN atau CTE (`WITH`), query dibungkus otomatis dalam subquery untuk operasi COUNT dan pagination.

```sql
SELECT * FROM (
  select a.id, a.name, b.category_name
  from item_product a inner join category b on b.category_id = a.category_id
) base_query
WHERE ...
ORDER BY ...
LIMIT 10 OFFSET 0
```

### Placeholder `${tableName}`

Placeholder `${tableName}` di dalam query diganti dengan `readSource` saat generation.

```json
{ "datatablesQuery": "select * from ${tableName} where status = 'active'" }
```

---

**Lihat juga**: [Field Sumber Data (RDF)](../../catalogs/rdf/data-source.md) · [Format Referensi File (RDF)](../../catalogs/rdf/file-reference.md) · [`features/`](../)
