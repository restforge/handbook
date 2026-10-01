# Catalogs

> Spec lengkap tiga definition format RESTForge: SDF, RDF, UDF.

## Tiga Definition Format

| Format | Kepanjangan | Tujuan | Audience |
|--------|-------------|--------|----------|
| **[SDF](./sdf/)** | Schema Definition File | Logical schema database | Database engineer, backend developer |
| **[RDF](./rdf/)** | Resource Definition File | Endpoint REST API definition | Backend developer, API integrator |
| **[UDF](./udf/)** | UI Definition File | Frontend page structure | Frontend developer, UI designer |

## Kapan Pakai Mana

| Skenario | Format |
|----------|--------|
| Mendeklarasikan tabel database baru dengan kolom, index, constraint, dan relasi | SDF |
| Mendeklarasikan endpoint REST CRUD beserta validasi, cache, dan audit | RDF |
| Mendeklarasikan halaman frontend CRUD beserta form, table, dan tombol aksi | UDF |
| Mengelola schema database existing (migrate, diff, apply, introspect) | SDF |
| Mengonversi tabel existing ke endpoint dengan setting default | RDF (di-generate dari SDF) |
| Mengonversi RDF ke UI default untuk mempercepat UI development | UDF (dikonversi dari RDF) |

## Alur Hubungan

![Definition File Flow](https://pub-6096676046da426597041567e928006f.r2.dev/handbook/definition-file.png)

Setiap format dapat ditulis secara manual atau di-generate dari format sebelumnya untuk konsistensi nama field.

## Versi Schema

Setiap catalog memiliki field `schemaVersion` yang mengikuti SemVer:
- **Patch**: tambah field opsional, tidak breaking
- **Minor**: tambah constraint/tipe/preset baru
- **Major**: breaking change struktur

Konsumen catalog (MCP server, LSP, AI agent) wajib cek `schemaVersion` sebelum parsing.
