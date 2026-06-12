# `payload validate`

> Memvalidasi file payload existing terhadap struktur tabel di database. Memberikan verdict pass/fail (OK / DRIFT / ERROR) tanpa detail per-column.

## Pattern

```
npx restforge payload validate --config=<FILE> [--table=<NAME>]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak | semua | Validasi hanya satu tabel spesifik |

## Logika Validasi

`payload validate` memakai rujukan tunggal dengan
`endpoint create` dan `payload diff`/`sync`: comparator audit-column-aware
`compareSchemaStrict`. Verdict yang dilaporkan:

| Status | Arti |
|--------|------|
| `[OK]` | Schema payload aligned dengan database |
| `[DRIFT]` | Ada perbedaan kolom, tipe, atau audit columns missing |
| `[ERROR]` | Tabel tidak ditemukan di database |

Kategori drift yang dideteksi:

| Kategori | Deskripsi |
|----------|-----------|
| Payload field missing from database | `fieldName` di payload tidak ada di DB |
| Type change | Tipe data di `fieldValidation` payload beda dengan DB |
| New database column | DB punya kolom yang tidak ada di payload (non-audit) |
| Audit column required but missing | `auditColumns` aktif (default `true`) tapi DB tidak punya kolom audit standar |

## Output Format

```
============================================================
SCHEMA VALIDATION - Payload vs Database
============================================================

  Payload directory : <path>
  Files to validate : <count>

------------------------------------------------------------

  [OK]    users.json (users)
  [DRIFT] visitors.json (visitors)
         4 audit column(s) required but missing
  [ERROR] legacy.json (legacy)
         Table "legacy" not found in database

------------------------------------------------------------

  Summary:
    Total  : 3 payload(s)
    OK     : 1
    Drift  : 1
    Error  : 1

  Advisories (informational, not errors):
    [i] visitors: has 1 foreign key(s) not surfaced as JOIN
        -> npx restforge payload sync --table=visitors --expand-fk

  Use 'npx restforge payload diff' for detailed comparison.
  Use 'npx restforge payload sync' to update payload files automatically.
```

## Saran Tindak Lanjut (Advisories)

Selain verdict, `payload validate` dapat menampilkan bagian **Advisories** berisi
saran tindak lanjut. Advisory bersifat informational, **bukan** error: tidak
memengaruhi verdict (`OK`/`DRIFT`/`ERROR`) maupun exit code. Bagian ini hanya
muncul bila ada saran yang relevan, dan dilewati untuk file ber-status `[ERROR]`
(tabel tidak ada di database).

| Advisory | Pemicu | Saran |
|----------|--------|-------|
| Foreign key belum di-surface sebagai JOIN | Tabel punya foreign key dan payload belum memakai JOIN (tidak ada `viewQuery`) | `npx restforge payload sync --table=<NAME> --expand-fk` |
| `is_active` ada tetapi `defaultScope` belum lengkap | Kolom `is_active` ada di `fieldName` tetapi `defaultScope` belum memuat `is_active` di `lookup` dan `read` | `npx restforge payload sync --table=<NAME>` |

Advisory foreign key dilewati bila payload sudah memakai konfigurasi JOIN (lihat
[`payload sync --expand-fk`](./sync.md#ekspansi-foreign-key---expand-fk)). Advisory
scope dilewati bila `defaultScope` sudah lengkap (lihat [Default Scope `is_active`
built-in](./sync.md#default-scope-is_active-built-in)).

## Exit Code

| Code | Arti |
|------|------|
| `0` | Semua payload OK |
| `1` | Ada drift atau error |

## Contoh (Examples)

```bat
npx restforge payload validate --config=db.env
npx restforge payload validate --config=db.env --table=users
```

## Hubungan dengan Command Lain

`validate` adalah **verdict mode**. Untuk inspect detail perbedaan kolom,
gunakan [`payload diff`](./diff.md). Untuk resolve drift secara otomatis,
gunakan [`payload sync`](./sync.md).

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
