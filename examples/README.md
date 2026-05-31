# Examples

> File contoh SDF, RDF, dan UDF yang siap di-copy-paste ke project baru.

## Struktur

| Folder | Isi |
|--------|-----|
| [`sdf/`](./sdf/) | File `.js` Schema Definition File untuk skenario umum |
| [`rdf/`](./rdf/) | File `.json` Resource Definition File untuk skenario umum |
| [`udf/`](./udf/) | File `.json` UI Definition File untuk skenario umum |

## Cara Pakai (Usage)

1. Copy file dari folder yang relevan ke folder konvensi di project:
   - SDF → `<project>/schema/`
   - RDF → `<project>/payload/`
   - UDF → `<project>/ui-payload/`
2. Sesuaikan nama tabel, kolom, atau page sesuai kebutuhan
3. Jalankan command generator yang sesuai (lihat [`commands/`](../commands/))

## Topik yang Akan Tersedia

File contoh akan ditambahkan secara bertahap. Skenario yang akan di-cover:

- SDF dasar dengan audit columns
- SDF dengan relasi parent-child (belongsTo, hasMany)
- SDF dengan check constraints
- RDF dasar CRUD
- RDF dengan custom query (datatables, lookup)
- RDF dengan fieldPolicy audit
- UDF dasar dengan form + datatables
- UDF dengan custom event handler
