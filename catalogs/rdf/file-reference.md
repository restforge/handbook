# Format Referensi File (`file:` Prefix)

Field SQL di payload (RDF dan dashboard) mendukung dua format penulisan: **inline SQL** atau **file reference**. Keduanya secara fungsional identik — generator membaca konten SQL dari file pada saat `endpoint create` atau `dashboard create` dijalankan, kemudian menyisipkan SQL tersebut sebagai template literal di dalam module `.js` yang di-generate. Tidak ada akses filesystem dari kode runtime hasil generate.

## Field yang Mendukung `file:` Prefix

### RDF (endpoint payload)

| Field | Lokasi di Payload | Konsumen di Generated Module |
|-------|-------------------|--------------------------------|
| `datatablesQuery` | top-level | method `getListQuery()` di model |
| `viewQuery` | top-level | method `getReadQuery()` di model |
| `exportQuery` | top-level | export handler |
| `masterDetail.detailConfig.detailQuery` | nested di bawah `masterDetail.detailConfig` | method `readComposite()` di model |
| `advancedQueries.<key>` | object value per key | template loader di constructor model |
| `nestedQueries.<key>` | array atau object value per key | builder query nested |
| `customQueries.<key>` | array atau object value per key | custom query handler |

### Dashboard payload

| Field | Lokasi di Payload | Konsumen di Generated Module |
|-------|-------------------|--------------------------------|
| `widgets[].query` | array of widget objects, field `query` | widget executor |
| `widgets[].queries.<key>` | object value per key di dalam widget | widget executor multi-query |

## Inline SQL

```json
{
    "datatablesQuery": "SELECT supplier_id, supplier_code, supplier_name FROM supplier"
}
```

| Karakteristik | Penjelasan |
|---------------|-----------|
| Format | String SQL langsung (umumnya dimulai dengan `SELECT`) |
| Use case | Query sederhana, single-line, tanpa JOIN kompleks |
| Trade-off | Kurang readable untuk query panjang; escape JSON string manual untuk karakter spesial |

## File Reference

```json
{
    "datatablesQuery": "file:query/supplier-datatables.sql"
}
```

| Karakteristik | Penjelasan |
|---------------|-----------|
| Format | String dimulai dengan prefix `file:` diikuti path relatif |
| Base path | Direktori payload (folder yang berisi file payload `.json`) |
| Konvensi folder | `payload/query/<file>.sql` untuk endpoint; `payload/query/<dashboard>/<widget>.sql` untuk dashboard widget |
| Use case | Query panjang, multi-line, dengan JOIN, formatting penuh SQL, syntax highlighting editor |
| Trade-off | File terpisah untuk dikelola; sebanding dengan keterbacaan untuk query non-trivial |

### Contoh File SQL

`payload/query/supplier-datatables.sql`:

```sql
SELECT a.supplier_id,
       a.supplier_code,
       a.supplier_name,
       a.email,
       a.phone,
       c.city_name
FROM supplier a
LEFT JOIN city c ON c.city_id = a.city_id
```

### Resolusi Path

Path setelah prefix `file:` di-resolve relatif terhadap direktori payload:

| Payload Path | Reference | Resolved File Path |
|---|---|---|
| `payload/supplier.json` | `file:query/supplier-datatables.sql` | `payload/query/supplier-datatables.sql` |
| `payload/stock-inbound.json` | `file:query/stock-inbound-detailquery.sql` | `payload/query/stock-inbound-detailquery.sql` |
| `payload/dashboard-inbound.json` | `file:query/dashboard-inbound/inbound-by-supplier.sql` | `payload/query/dashboard-inbound/inbound-by-supplier.sql` |

## Perilaku Generator

Saat `endpoint create` atau `dashboard create` dijalankan:

1. Payload `.json` dibaca dari `payload/<name>.json`.
2. Setiap field SQL yang berformat `file:...` di-resolve ke path absolut, isi file `.sql` dibaca.
3. Konten SQL disisipkan sebagai template literal di module `.js` yang di-generate.
4. File `.js` hasil generate berdiri sendiri tanpa dependency pada file `.sql` saat runtime.

Output untuk inline SQL dan file reference identik secara struktural — pilihan format hanya mempengaruhi pengalaman authoring, bukan runtime behavior atau performa.

### Konsistensi Lintas Dialect

Behavior identik untuk keempat dialect database (PostgreSQL, MySQL, Oracle, SQLite). Generator menyisipkan SQL apa adanya ke template literal; transformasi sintaks dialect-specific (misalnya placeholder `$1` → `?` untuk MySQL, `$1` → `:1` untuk Oracle) dilakukan oleh helper konversi runtime, bukan oleh mekanisme `file:` reference.

### Komposisi Output Module

Module `.js` hasil generate **tidak memuat**:
- Pemanggilan `fs.readFileSync` di method request handler.
- Pemanggilan `path.join(__dirname, 'query', ...)` untuk lokasi runtime.
- Dependency pada folder `src/models/<project>/query/` saat runtime.

Module sepenuhnya self-contained: SQL ter-embed sebagai string literal dalam method seperti `getListQuery()`, `getReadQuery()`, dan `readComposite()`.

## Defensive Validation

Generator memvalidasi bahwa setiap field SQL yang disebutkan di table "Field yang Mendukung `file:` Prefix" tidak berisi string `"file:..."` saat siap di-render ke template. Apabila terjadi anomali (mis. konfigurasi processor yang menyimpang), generator akan throw error:

```
Codegen invariant violation: <fieldName> still contains "file:" reference (<value>).
Expected expanded SQL string.
```

Pesan ini mengindikasikan bahwa expansion `file:` tidak terjadi sesuai harapan untuk field tersebut — periksa konfigurasi processor atau struktur payload.

## Tips Authoring

- Letakkan file `.sql` di sub-folder `payload/query/` untuk kerapian struktur.
- Untuk dashboard yang punya multiple widget, gunakan sub-folder per dashboard: `payload/query/<dashboard-name>/<widget>.sql`.
- Tidak perlu meng-quote string SQL secara manual (escape JSON) — gunakan format file reference untuk query multi-line.
- Validasi syntax SQL dilakukan saat `endpoint create` (opsional di-skip via `--skip-sql-validation`).

---

**Lihat juga**: [`rdf/`](./) · [`master-detail.md`](./master-detail.md) · [`catalogs/`](../) · [`README`](../../README.md)
