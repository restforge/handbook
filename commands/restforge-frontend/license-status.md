# `restforge-designer license status`

> Tampilkan status license aktif: source resolusi, storage backend, dan hasil validasi server.

## Sintaks

```bash
restforge-designer license status [OPTIONS]
```

## Flag

| Flag | Tipe | Keterangan |
|------|------|-----------|
| `--plugins-dir <PATH>` | string | Override folder plugins |
| `--license <KEY>` | string | License key override per-invocation (jarang dipakai di status) |

## Contoh Penggunaan

### Status License Aktif

```bash
restforge-designer license status
```

Output dengan license aktif:

```
┌─────────────────────┬───────────────────────────────────────┐
│ RESTForge Designer License Status                           │
├─────────────────────┼───────────────────────────────────────┤
│ Field               │ Value                                 │
├─────────────────────┼───────────────────────────────────────┤
│ Active key          │ ABCD-1234-EFGH-5678                   │
│ Resolved from       │ encrypted storage                     │
│ Encrypted storage   │ OS keyring                            │
│ OS keyring          │ yes                                   │
└─────────────────────┴───────────────────────────────────────┘

[Checking validation status...]

╭─ Validation Result ─────────────────────────╮
│ Validation : valid                          │
│ Product    : RESTForge Designer             │
│ Expires    : 2026-12-31                     │
╰─────────────────────────────────────────────╯
```

Output tanpa license:

```
┌─────────────────────┬───────────────────────────────────────┐
│ Field               │ Value                                 │
├─────────────────────┼───────────────────────────────────────┤
│ Active key          │ not configured                        │
│ Resolved from       │ none                                  │
│ Encrypted storage   │ none                                  │
│ OS keyring          │ yes                                   │
└─────────────────────┴───────────────────────────────────────┘

Run `restforge-designer activate --key XXXX-XXXX-XXXX-XXXX` to configure a license.
```

## Behavior

Verb `license status`:

1. **Resolve license key** via 4-priority chain:
   1. `--license` flag (jika dipakai)
   2. Encrypted storage (OS keyring atau encrypted file fallback)
   3. File `.env` field `LICENSE=`
   4. OS env var `RESTFORGE_DESIGNER_LICENSE_KEY`
2. **Identify source** yang menang di chain (tampilkan di field `Resolved from`)
3. **Detect storage backend**: OS keyring atau encrypted file fallback
4. **Live validation** ke license server resmi (jika ada key), memakai memory + disk cache untuk hindari spam server
5. **Tampilkan tabel** + panel hasil validasi

## Field di Output

### Tabel Status

| Field | Possible Value | Arti |
|-------|----------------|------|
| `Active key` | `XXXX-XXXX-XXXX-XXXX` / `not configured` | License key yang resolved, atau tidak ada |
| `Resolved from` | `encrypted storage` / `environment variable (RESTFORGE_DESIGNER_LICENSE_KEY)` / `.env file (legacy)` / `none` | Source mana di chain yang menang |
| `Encrypted storage` | `OS keyring` / `encrypted file` / `none` | Backend storage yang dipakai |
| `OS keyring available` | `yes` / `no (using encrypted file fallback)` | Apakah OS keyring tersedia |

### Panel Validation Result

| Field | Possible Value | Arti |
|-------|----------------|------|
| `Validation` | `valid` / `valid (cached)` / `Validation: <error message>` | Hasil call ke server (atau dari cache) |
| `Product` | `RESTForge Designer` | Nama produk yang license cover |
| `Expires` | `YYYY-MM-DD` | Tanggal expiry license |

Suffix `(cached)` muncul jika hasil validasi diambil dari cache lokal (tidak hit server). Cache TTL ditentukan oleh server.

## Identify Source Logic

Source ditentukan dengan membandingkan key yang resolved dengan key di setiap storage:

| Skenario | `Resolved from` |
|----------|-----------------|
| Key di encrypted storage = resolved | `encrypted storage` |
| Tidak ada di storage, key di env var = resolved | `environment variable (RESTFORGE_DESIGNER_LICENSE_KEY)` |
| Tidak ada di storage/env var, ada di `.env` | `.env file (legacy)` |
| Semua kosong | `none` |

`.env file (legacy)` ditandai legacy karena RESTForge Designer (versi baru) merekomendasikan encrypted storage. File `.env` masih didukung tetapi dianggap pola lama. Lihat [`license-migrate.md`](./license-migrate.md) untuk pindah ke encrypted storage.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Status berhasil ditampilkan (terlepas dari hasil validasi) |
| `1` | Live validation gagal (server unreachable, license invalid, expired) — kecuali jika tidak ada key (graceful exit 0) |

## Use Case

| Skenario | Behavior |
|----------|----------|
| Mau cek "apakah license sudah ter-setup" | Lihat field `Active key` — kalau `not configured`, perlu `activate` |
| Mau cek "kapan license expire" | Lihat panel `Expires` |
| Mau cek "mengapa license tidak terdeteksi" | Lihat `Resolved from` untuk tahu chain mana yang dipakai |
| Mau cek "apakah OS keyring berfungsi" | Lihat `OS keyring available` |
| Debug "kenapa generate tidak jalan" | Mulai dari `license status` untuk pastikan license valid |

---

**Lihat juga**: [`README`](./README.md) · [`activate.md`](./activate.md) · [`deactivate.md`](./deactivate.md) · [`license-migrate.md`](./license-migrate.md)
