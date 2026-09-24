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
| `--auto-create-db` | `false` | Khusus postgres/mysql: otomatis buat database jika belum ada (skip konfirmasi). Berguna untuk CI/CD |

## Behavior Database Tidak Ada (PostgreSQL/MySQL)

Sebelum menerapkan DDL, `schema migrate` melakukan pre-check existence database target. Apabila database belum ada pada server PostgreSQL atau MySQL, command akan menawarkan pembuatan database. Pemilihan jalur ditentukan oleh konteks eksekusi:

| Konteks | Behavior |
|---------|----------|
| Interactive (TTY) tanpa flag | Tampilkan prompt: `Do you want to create a new database? (y/N):` |
| Interactive (TTY) dengan `--auto-create-db` | Langsung create tanpa prompt |
| Non-interactive (CI/CD) tanpa flag | Skip pembuatan, tampilkan hint: `To create the database automatically in non-interactive mode, add the --auto-create-db flag.` |
| Non-interactive (CI/CD) dengan `--auto-create-db` | Langsung create tanpa prompt |

Sebelum prompt ditampilkan, command mencetak baris informasi: `Database '<DB_NAME>' was not found on the postgresql server.` (atau `mysql` sesuai dialect).

Setelah database berhasil dibuat, command mencetak `[OK] Database '<DB_NAME>' created successfully.` lalu menampilkan instruksi `Please re-run the command:` diikuti contoh perintah re-run. Pendekatan create-then-exit dipilih agar state migrate dimulai dari kondisi bersih, bukan auto-continue dari state yang baru berubah.

Mekanisme pembuatan database:

- **PostgreSQL**: koneksi ke database administratif `postgres` menggunakan credential yang sama, lalu eksekusi `CREATE DATABASE <DB_NAME>`
- **MySQL**: koneksi tanpa parameter database, lalu eksekusi `CREATE DATABASE <DB_NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`
- **SQLite**: tidak relevan, file database otomatis dibuat saat koneksi pertama
- **Oracle**: tidak relevan, menggunakan `SERVICE_NAME` bukan database

User pada `DB_USER` harus memiliki privilege `CREATE DATABASE`. Apabila privilege tidak mencukupi, command akan menampilkan pesan: `Error: User '<name>' does not have privilege to create a database.` diikuti saran `Please contact the database administrator to create the database '<DB_NAME>' manually.`

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
