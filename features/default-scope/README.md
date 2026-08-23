# Default Scope

Default scope berfungsi untuk menyisipkan filter `WHERE` secara otomatis ke endpoint `lookup` dan `read`, sehingga record non-aktif tidak ikut tampil tanpa perlu menulis kondisi yang sama berulang kali. Aturan filter ditulis sekali di payload, lalu berlaku konsisten di semua request.

Halaman ini berisi panduan tugas. Untuk ringkasan teknis yang padat, lihat [`catalogs/rdf/default-scope.md`](../../catalogs/rdf/default-scope.md).

> **Catatan:** Tanpa properti `defaultScope`, generator memperlakukan payload seperti sebelum fitur ini ada. Fitur ini aman ditambahkan ke project yang sudah berjalan karena payload lama tidak terpengaruh.

---

## Menambahkan Default Scope di Payload (Adding the Scope)

Untuk mendeklarasikan default scope di dalam payload, tulis properti `defaultScope` di level root payload, sejajar dengan `tableName` dan `action`. Setiap key adalah nama action yang difilter, dan nilainya pasangan kolom dengan nilai yang harus dipenuhi.

```json
{
    "tableName": "category",
    "primaryKey": "category_id",
    "fieldName": ["category_id", "category_code", "category_name", "is_active"],
    "defaultScope": {
        "lookup": { "is_active": true },
        "read": { "is_active": true }
    },
    "action": { "datatables": true, "lookup": true, "read": true, "first": true }
}
```

Generator menerjemahkan konfigurasi ini menjadi `WHERE` clause dan menanamkannya ke SQL action `lookup` dan `read`. Pada endpoint lookup dengan data lima kategori yang dua di antaranya non-aktif, hasilnya menyusut menjadi tiga.

```
GET /api/dbinv-postgres/category/lookup?search=
```

```json
{
  "success": true,
  "count": 3,
  "data": [
    { "id": "uuid-1", "text": "ELK - Elektronik" },
    { "id": "uuid-2", "text": "MKN - Makanan" },
    { "id": "uuid-4", "text": "FRN - Furniture" }
  ]
}
```

> **Catatan:** Untuk kolom `is_active`, konfigurasi ini tidak perlu ditulis secara manual. `payload generate` menyisipkannya secara otomatis, dan `payload sync` menjaganya tetap selaras saat kolom `is_active` ditambah atau dihapus. Sinkronisasi bersifat presisi dan hanya menyentuh key `is_active`, sehingga filter manual untuk kolom lain seperti `show_in_store` atau `tenant_id` tidak ikut hilang.

---

## Memilih Action yang Difilter (Choosing Actions)

Default scope hanya menyentuh action yang menampilkan data untuk pengguna umum, dan sengaja membiarkan action milik admin tetap melihat seluruh data.

| Action | Difilter? | Alasan |
|--------|:---------:|--------|
| `lookup` | Ya | Dropdown sebaiknya hanya menawarkan pilihan yang masih aktif |
| `read` | Ya | Halaman list untuk pengguna umum cukup menampilkan record aktif |
| `datatables` | Tidak | Admin perlu melihat seluruh data, termasuk yang non-aktif, untuk mengelolanya |
| `first` | Tidak | Pengambilan satu record by ID harus selalu berhasil, misalnya saat membuka form Edit record lama |

Tuliskan hanya key untuk action yang ingin dibatasi, dan filter antar action boleh berbeda. Pada contoh berikut, dropdown menyaring berdasarkan status aktif, sedangkan halaman list menambahkan syarat lain.

```json
"defaultScope": {
    "lookup": { "is_active": true },
    "read": { "is_active": true, "show_in_store": true }
}
```

Nilai filter berupa boolean, string, atau angka, sesuai kebutuhan kolomnya.

| Tipe nilai | Contoh | Maksudnya |
|------------|--------|-----------|
| boolean | `{ "is_active": true }` | Record yang kolomnya bernilai benar |
| string | `{ "status": "active" }` | Record yang kolomnya cocok dengan teks tersebut |
| angka | `{ "level": 1 }` | Record yang kolomnya bernilai angka tersebut |

---

## Menggabungkan dengan Filter Pengguna (Combining with User Filters)

Default scope tidak menggantikan filter dari pengguna, melainkan menambahkannya dengan `AND`. Scope selalu berlaku, dan filter pengguna bekerja di dalam batas tersebut.

Misalnya pengguna mengirim filter untuk mencari kode yang mengandung huruf "K".

```json
{ "where": [ { "key": "category_code", "operator": "LIKE", "value": "%K%" } ] }
```

Query efektif yang dijalankan menjadi `WHERE is_active = true AND (category_code LIKE '%K%')`. Record non-aktif tetap tersaring meski cocok dengan pola pencarian.

> **Catatan:** Pada endpoint `read`, scope juga memengaruhi `total_records`. Jumlah paginasi dihitung dari data yang sudah terfilter, sehingga lima record dengan dua non-aktif menghasilkan `total_records` bernilai tiga.

---

## Memfilter di Processor dengan `.active()` (Filtering in Processors)

Default scope hanya berlaku untuk endpoint hasil generate. Di dalam processor, query builder menyediakan method `.active()` sebagai cara singkat menyaring record aktif.

```javascript
const categories = await db.table('category').active().get();

const suppliers = await db.table('supplier').active().where({ city: 'JAKARTA' }).get();

const items = await db.table('item_product').active('status_active').get();
```

Method `.active()` setara dengan `.where({ is_active: true })` dan mengembalikan instance query builder, sehingga bisa dirangkai dengan `.where()`, `.orderBy()`, `.limit()`, `.first()`, maupun `.count()`.

> **Catatan:** Kolom default-nya `is_active`. Berikan nama kolom lain sebagai argumen bila status disimpan di kolom berbeda, misalnya `.active('status_active')`.

---

## Memilih Antara Payload dan Processor (Payload vs Processor)

Kedua pendekatan menyaring record aktif, tetapi dipakai pada situasi yang berbeda.

| Aspek | Payload `defaultScope` | Query builder `.active()` |
|-------|------------------------|---------------------------|
| Sasaran | Endpoint CRUD hasil generate | Kode custom di processor |
| Cara menulis | Deklaratif di payload JSON | Imperatif di JavaScript |
| Kapan menempel | Saat generate, jadi bagian SQL | Saat runtime, sebagai query biasa |
| Cakupan | Per action `lookup` atau `read` | Per query sesuai kebutuhan |

Gunakan `defaultScope` bila endpoint `lookup` atau `read` memang harus selalu menyaring kolom tertentu dan filternya bersifat tetap. Gunakan `.active()` bila menulis logic yang tidak di-generate, atau bila filter perlu menyesuaikan kondisi, misalnya admin melihat semua data sedangkan pengguna biasa hanya yang aktif.

---

## Perilaku di Berbagai Database (Behavior Across Databases)

Setiap database menyimpan boolean dengan caranya sendiri, dan generator menyesuaikan format SQL berdasarkan database project. Filter `{ "is_active": true }` cukup ditulis satu kali.

| Database | Tipe kolom | SQL yang dihasilkan |
|----------|-----------|---------------------|
| PostgreSQL | BOOLEAN | `WHERE a.is_active = true` |
| MySQL | VARCHAR | `WHERE is_active = 'true'` |
| Oracle | VARCHAR2 | `WHERE is_active = 'true'` |

> **Catatan:** Prinsip inilah inti RESTForge. Satu payload menghasilkan perilaku yang setara di semua database, dan perbedaan dialek ditangani oleh generator, bukan oleh developer.

---

## Menangani Konfigurasi Keliru (Handling Misconfiguration)

Generator memeriksa `defaultScope` saat membaca payload, sehingga kesalahan terdeteksi lebih awal, bukan saat endpoint sudah berjalan.

| Kondisi | Reaksi generator |
|---------|------------------|
| Kolom filter tidak terdaftar di `fieldName` | Berhenti dengan error yang menyebut kolomnya |
| Nilai filter bukan boolean, string, atau angka | Berhenti dengan error tipe |
| Key action tidak dikenal, misalnya `datatables` | Lanjut dengan warning, key diabaikan |

```
Error: defaultScope.lookup references column 'status' which is not in fieldName of category.json
Warning: defaultScope key 'datatables' in category.json is not recognized. Only 'lookup' and 'read' are supported.
```

> **Catatan:** Hanya `lookup` dan `read` yang bisa diberi default scope, karena hanya kedua action itu yang ditujukan untuk menyaring data bagi pengguna umum.

---

## Referensi Cepat (Quick Reference)

| Properti | Nilai |
|----------|-------|
| Lokasi konfigurasi | `defaultScope` di level root payload JSON |
| Action yang difilter | `lookup`, `read` |
| Action yang tidak difilter | `datatables`, `first` |
| Bentuk nilai filter | Pasangan kolom dan nilai berupa boolean, string, atau angka |
| Gabungan dengan filter pengguna | Selalu disertakan dengan `AND` |
| Auto untuk `is_active` | Dikelola otomatis oleh `payload generate` dan `payload sync` |
| Padanan di processor | Method `.active()` di query builder |
| Database | PostgreSQL, MySQL, Oracle, SQLite |

**Lihat juga**: [Referensi RDF default scope](../../catalogs/rdf/default-scope.md) · [`catalogs/`](../../catalogs/) · [`features/`](../)
