# `validate`

> Memvalidasi file config database tanpa menjalankan server. Berguna untuk preflight check sebelum deploy.

Validasi mencakup koneksi database (PostgreSQL/MySQL/Oracle/SQLite), Redis (jika fitur cache/idgen/lock/idempotency/rate-limit/live-sync aktif), Kafka (jika `KAFKA_ENABLED`), serta status license.

## Pattern

```
npx restforge validate --config=<FILE> [--auto-create-db]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config yang divalidasi (mis. `db-connection.env`) |
| `--auto-create-db` | Tidak | `false` | Khusus postgres/mysql: otomatis buat database jika belum ada (lewati konfirmasi). Berguna untuk CI/CD dan Docker. Tidak berlaku untuk sqlite/oracle |

## Behavior Database Tidak Ada (PostgreSQL/MySQL)

Apabila target database (`DB_NAME`) belum ada pada server PostgreSQL atau MySQL, command `validate` akan otomatis menawarkan pembuatan database. Pemilihan jalur ditentukan oleh konteks eksekusi:

| Konteks | Behavior |
|---------|----------|
| Interactive (TTY) tanpa flag | Tampilkan prompt: `Do you want to create a new database? (y/N):` |
| Interactive (TTY) dengan `--auto-create-db` | Langsung create tanpa prompt |
| Non-interactive (CI/CD) tanpa flag | Lewati pembuatan, tampilkan hint: `To create the database automatically in non-interactive mode, add the --auto-create-db flag.` |
| Non-interactive (CI/CD) dengan `--auto-create-db` | Langsung create tanpa prompt |

Sebelum prompt ditampilkan, command mencetak baris informasi: `Database '<DB_NAME>' was not found on the postgresql server.` (atau `mysql` sesuai dialect).

Setelah database berhasil dibuat, command mencetak `[OK] Database '<DB_NAME>' created successfully.` lalu menampilkan instruksi `Please re-run the command:` diikuti contoh perintah re-run. Command exit dengan code `0`. State berubah karena pembuatan database, sehingga validasi ulang diperlukan untuk konfirmasi seluruh komponen.

Mekanisme pembuatan database:

- **PostgreSQL**: koneksi ke database administratif `postgres` menggunakan credential yang sama, lalu eksekusi `CREATE DATABASE <DB_NAME>`
- **MySQL**: koneksi tanpa parameter database, lalu eksekusi `CREATE DATABASE <DB_NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`
- **SQLite**: tidak relevan, file database otomatis dibuat saat koneksi pertama
- **Oracle**: tidak relevan, menggunakan `SERVICE_NAME` bukan database

User pada `DB_USER` harus memiliki privilege `CREATE DATABASE`. Apabila privilege tidak mencukupi, command akan menampilkan pesan: `Error: User '<name>' does not have privilege to create a database.` diikuti saran `Please contact the database administrator to create the database '<DB_NAME>' manually.`

## Contoh (Examples)

```bat
:: Validasi config standar
npx restforge validate --config=config/db-connection.env

:: Validasi pada CI/CD dengan auto-create database
npx restforge validate --config=config/db-connection.env --auto-create-db
```

---

**Lihat juga**: [`runtime/`](./) · [`schema/migrate`](../schema/migrate.md) · [`commands/`](../) · [`README`](../../../README.md)
