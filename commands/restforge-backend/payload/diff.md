# `payload diff`

> Menampilkan perbedaan kolom (column-level differences) antara payload JSON dan tabel database secara detail.

## Pattern

```
npx restforge payload diff --config=<FILE> [--table=<NAME>]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak | semua | Diff hanya satu tabel spesifik |

## Logika Diff

`payload diff` memakai rujukan tunggal dengan
`endpoint create` dan `payload validate`/`sync`: comparator audit-column-aware
`compareSchemaStrict`. Berbeda dengan `validate` yang hanya menampilkan verdict,
`diff` menampilkan **detail per-column** untuk setiap kategori drift.

## Notasi Detail

| Notasi | Arti |
|--------|------|
| `[-] <col>` | Kolom ada di payload tapi tidak di database |
| `[~] <col>` | Tipe kolom berbeda antara payload `fieldValidation` dan database |
| `[+] <col>` | Kolom ada di database tapi tidak di payload |
| `[+] created_at, ...` | Kolom audit diwajibkan oleh `auditColumns=true` tapi tidak di database |

Kolom yang sengaja dihapus dari `fieldName` tidak ditampilkan sebagai `[+]`. Aturannya sama dengan [`payload validate`](./validate.md#logika-validasi).

Query SQL yang gagal dijalankan ke database ditampilkan di bawah detail kolom beserta pesan database. Contoh berikut menunjukkan kolom `email` yang sudah di-drop dari tabel tetapi masih dipilih file SQL:

```
  [DRIFT] users.json (users)
         1 payload field(s) missing from database
         [-] email    (in payload, not in database)
         Query check failed against the database:
           [!] datatablesQuery: 42703: column a.email does not exist
```

Aturan status untuk query yang gagal sama dengan [`payload validate`](./validate.md#query-sql-yang-gagal).

## Output Format

```
============================================================
SCHEMA DIFF - Payload vs Database
============================================================

  Payload directory : <path>
  Files to compare  : <count>

------------------------------------------------------------

  [DRIFT] visitors.json (visitors)
         4 audit column(s) required but missing
         [+] created_at, created_by, updated_at, updated_by
             (required by auditColumns=true, not in database)

  [DRIFT] orders.json (orders)
         1 payload field(s) missing from database, 1 type change(s), 1 new database column(s) not in payload
         [-] obsolete_field    (in payload, not in database)
         [~] qty    (type: varchar -> integer)
         [+] new_status    (varchar, nullable) (in database, not in payload)

  [OK]    users.json (users)
         Schema is in sync

------------------------------------------------------------

  Summary:
    Total  : 3 payload(s)
    OK     : 1
    Drift  : 2

  Use 'npx restforge payload sync' to update payload files automatically.
```

## Exit Code

| Code | Arti |
|------|------|
| `0` | Semua payload OK |
| `1` | Ada drift atau error |

## Contoh

```bat
npx restforge payload diff --config=db.env
npx restforge payload diff --config=db.env --table=orders
```

## Hubungan dengan Command Lain

`diff` adalah **inspector mode**. Untuk verdict ringkas saja, gunakan
[`payload validate`](./validate.md). Untuk resolve drift secara otomatis,
gunakan [`payload sync`](./sync.md).

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
