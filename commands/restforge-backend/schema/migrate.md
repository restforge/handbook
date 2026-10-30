# `schema migrate`

> Apply schema definition ke database melalui driver dialect (load → validate → generate DDL → execute).

## Pattern

```
npx restforge schema migrate --schema-path=<PATH> --config=<FILE> [options]
```

## Flag Wajib

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--schema-path <PATH>` | - | Path file atau folder schema |
| `--config <FILE>` | - | File config database |

## Flag Opsional

| Flag | Default | Keterangan |
|------|---------|-----------|
| `--drop <BOOL>` | `false` | Drop tabel sebelum re-create |
| `--dry-run` | `false` | Preview DDL tanpa menjalankan ke database |
| `--max-name-length <N>` | per dialect | Batas panjang identifier (override default) |
| `--auto-create-db` | `false` | Khusus postgres/mysql: otomatis buat database jika belum ada (skip konfirmasi), lalu migrasi dilanjutkan pada proses yang sama. Berguna untuk CI/CD |

## Behavior Database Tidak Ada (PostgreSQL/MySQL)

Sebelum menerapkan DDL, `schema migrate` melakukan pre-check existence database target. Apabila database belum ada pada server PostgreSQL atau MySQL, command akan menawarkan pembuatan database. Pemilihan jalur ditentukan oleh konteks eksekusi:

| Konteks | Behavior |
|---------|----------|
| Interactive (TTY) tanpa flag | Tampilkan prompt: `Do you want to create a new database? (y/N):` |
| Interactive (TTY) dengan `--auto-create-db` | Langsung create tanpa prompt |
| Non-interactive (CI/CD) tanpa flag | Skip pembuatan, tampilkan hint: `To create the database automatically in non-interactive mode, add the --auto-create-db flag.` |
| Non-interactive (CI/CD) dengan `--auto-create-db` | Langsung create tanpa prompt |

Sebelum prompt ditampilkan, command mencetak baris informasi: `Database '<DB_NAME>' was not found on the postgresql server.` (atau `mysql` sesuai dialect).

Setelah database berhasil dibuat, command mencetak `[OK] Database '<DB_NAME>' created successfully.` lalu langsung menerapkan DDL ke database baru pada proses yang sama, sehingga command tidak perlu dijalankan ulang. Exit code mengikuti hasil penerapan DDL.

Mekanisme pembuatan database:

- **PostgreSQL**: koneksi ke database administratif `postgres` menggunakan credential yang sama, lalu eksekusi `CREATE DATABASE <DB_NAME>`
- **MySQL**: koneksi tanpa parameter database, lalu eksekusi `CREATE DATABASE <DB_NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`
- **SQLite**: tidak relevan, file database otomatis dibuat saat koneksi pertama
- **Oracle**: tidak relevan, menggunakan `SERVICE_NAME` bukan database

User pada `DB_USER` harus memiliki privilege `CREATE DATABASE`. Apabila privilege tidak mencukupi, command akan menampilkan pesan: `Error: User '<name>' does not have privilege to create a database.` diikuti saran `Please contact the database administrator to create the database '<DB_NAME>' manually.`

## Project Multi-Tenant

Project yang sudah menjalankan `project tenant --activate` memiliki `TENANT_MODE=multi` di config. Pada project seperti ini, `schema migrate` memberi scope tenant pada SDF tabel baru, yaitu tabel yang belum ada di database dan SDF-nya belum punya field `tenant_id`. Tabel baru tidak membutuhkan `project tenant --activate` lagi.

Sebelum DDL disusun, field `tenant_id` dan index-nya ditambahkan ke SDF, lalu unique diubah menjadi unique gabungan berawalan `tenant_id`. SDF lama diarsipkan ke `.restforge/archive/` sebelum file ditulis ulang. Tabel kemudian dibuat langsung dengan kolom `tenant_id` dan unique per tenant, sehingga tidak ada data yang perlu diisi tenant-nya.

```
TENANT_MODE=multi: new table(s) tenant-scoped (SDF rewritten, previous file archived in .restforge/archive/):
  [+tenant_id +index] supplier → schema\supplier.js
    supplier: UNIQUE (supplier_code) -> UNIQUE (tenant_id, supplier_code)
```

Dengan `--dry-run`, rencana dan DDL ditampilkan tanpa menulis SDF. Aturan pengecualian sama dengan `project tenant --activate`. Tabel registry tenant dan tabel di schema `auth` dilewati, sedangkan unique yang dirujuk foreign key, unique pada tabel soft delete, dan unique username login tetap berlaku untuk seluruh tabel.

SDF yang sudah punya field `tenant_id` tidak diubah. Tabel yang sudah ada di database tanpa kolom `tenant_id` juga tidak diubah, karena baris di dalamnya perlu diisi tenant lewat `project tenant --activate`. Untuk tabel seperti ini, output menampilkan peringatan berikut.

```
[warning] TENANT_MODE=multi: table legacy_note already exists in the database and its SDF has no tenant_id field. The table is left unchanged; run `npx restforge project tenant --activate` to add the column and assign existing rows to a tenant.
```

Dengan `--drop`, semua tabel dibuat ulang sehingga diperlakukan sebagai tabel baru. Nama field mengikuti `TENANT_COLUMN` di config bila nilainya bukan `tenant_id`.

## Relasi Melingkar

Dua tabel atau lebih bisa saling merujuk. Contohnya, `warehouse` wajib punya lokasi stok default di `location`, sedangkan `location` boleh menunjuk gudang pemiliknya. Pada pasangan seperti ini tidak ada tabel yang bisa dibuat lebih dulu bila semua foreign key ditulis di dalam `CREATE TABLE`.

`schema migrate` memutus lingkaran dengan menunda satu foreign key yang boleh kosong. Tabel di dalam lingkaran dibuat tanpa foreign key tersebut, lalu foreign key ditambahkan dengan `ALTER TABLE` setelah semua tabel ada. Nama constraint sama dengan foreign key biasa, sehingga `schema validate --config` dan `schema diff` tidak melihat perbedaan.

```sql
-- ADD FOREIGN KEY section (circular relations)
ALTER TABLE location ADD CONSTRAINT fk_location_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouse (warehouse_id) ON DELETE RESTRICT;
```

Foreign key yang ditunda dipilih dari relasi `belongsTo` yang kolomnya tidak `notnull`, dan pilihannya selalu sama untuk SDF yang sama. Data pada pasangan di atas diisi berurutan: lokasi dibuat dengan `warehouse_id` kosong, gudang dibuat dengan lokasi tersebut, lalu `warehouse_id` pada lokasi diisi.

Perilaku per dialect berbeda pada beberapa hal berikut.

- SQLite tidak mendukung `ALTER TABLE ADD CONSTRAINT`, tetapi menerima foreign key ke tabel yang dibuat belakangan. Pada SQLite, foreign key tetap ditulis di dalam `CREATE TABLE` dan hanya urutan tabelnya yang menyesuaikan.
- Tanpa `--drop`, `ALTER TABLE` dilewati bila constraint sudah ada, sehingga migrate bisa dijalankan ulang pada database yang sama.
- Dengan `--drop`, MySQL mematikan `FOREIGN_KEY_CHECKS` selama tabel dihapus, sedangkan SQLite menunda pengecekan foreign key sampai migrate selesai. PostgreSQL dan Oracle sudah menghapus constraint terkait saat tabel dihapus.

Lingkaran yang semua foreign key-nya `notnull` tetap ditolak, karena tidak ada baris yang bisa diisi lebih dulu. Pesan error menyebut tabel dan relasi di dalam lingkaran.

```
Error: DDL generation failed: Circular dependency detected among tables: location, warehouse. The cycle cannot be broken because these foreign keys are NOT NULL: location.warehouse -> warehouse, warehouse.default_stock_location -> location. Make at least one foreign key in the cycle nullable.
```

Perilaku yang sama berlaku untuk `schema generate-ddl`. [`schema validate`](./validate.md) menandai lingkaran yang bisa diputus dengan warning `circular-relation-nullable`.

## Contoh

```bat
:: Dry run untuk preview
npx restforge schema migrate --schema-path=./schema --config=db.env --dry-run

:: Apply ke database
npx restforge schema migrate --schema-path=./schema --config=db.env

:: Apply dengan auto-create-db untuk CI/CD
npx restforge schema migrate --schema-path=./schema --config=db.env --auto-create-db
```

---

**Lihat juga**: [`schema/`](./) · [`runtime/validate`](../runtime/validate.md) · [`commands/`](../) · [`README`](../../../README.md)
