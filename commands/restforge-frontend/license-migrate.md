# `restforge-designer license migrate`

> Migrasi license key dari file `.env` ke encrypted storage (one-time migration).

## Sintaks

```bash
restforge-designer license migrate [OPTIONS]
```

## Flag

| Flag | Tipe | Default | Keterangan |
|------|------|---------|-----------|
| `--no-validate` | flag | `false` | Skip server validation sebelum migrasi (offline migration) |
| `--force` | flag | `false` | Overwrite encrypted key existing dengan nilai dari `.env` |
| `--keep-env` | flag | `false` | Jangan hapus baris `LICENSE=` dari `.env` setelah migrasi |
| `--plugins-dir <PATH>` | string | auto-detect | Override folder plugins |
| `--license <KEY>` | string | dari `.env` | License key override per-invocation |

## Contoh Penggunaan

### Migrasi Standar

```bash
restforge-designer license migrate
```

Behavior:

1. Baca `LICENSE=` dari `.env` di working directory
2. Validasi ke server
3. Simpan ke encrypted storage
4. Hapus baris `LICENSE=` dari `.env`

Output sukses:

```
╭─ License Migration Complete ────────────────────────╮
│ Source       : .env (LICENSE=ABCD-1234-EFGH-5678)   │
│ Target       : OS keyring                           │
│ Validation   : successful                           │
│ .env cleanup : LICENSE line removed                 │
╰─────────────────────────────────────────────────────╯
```

### Migrasi Offline (Skip Validasi Server)

```bash
restforge-designer license migrate --no-validate
```

### Overwrite Encrypted Key Existing

Jika sudah ada key di encrypted storage dan ingin replace dengan key di `.env`:

```bash
restforge-designer license migrate --force
```

Tanpa `--force`, command gagal dengan pesan:

```
╭─ Migration Aborted ──────────────────────────────────────────────╮
│ An encrypted license is already stored: ZZZZ-9999-YYYY-8888      │
│ Use --force to overwrite, or run `restforge-designer deactivate` │
│ first.                                                           │
╰──────────────────────────────────────────────────────────────────╯
```

### Migrasi Tanpa Hapus dari `.env`

Berguna jika `.env` masih dipakai sebagai backup/audit:

```bash
restforge-designer license migrate --keep-env
```

Baris `LICENSE=` tetap di `.env` setelah migrasi. Namun chain resolusi akan mengembalikan key dari encrypted storage (priority 2) sebelum sampai ke `.env` (priority 3).

## Behavior

Verb `license migrate`:

1. **Resolve license key** dari `.env` (priority 3 di chain)
   - Jika `.env` tidak ada atau tidak punya `LICENSE=`, gagal dengan pesan tidak ada key untuk migrasi
2. **Validasi format** pattern `XXXX-XXXX-XXXX-XXXX`
3. **Cek existing encrypted storage**:
   - Jika sudah ada key sama → no-op (idempotent), tetap hapus dari `.env` jika tidak `--keep-env`
   - Jika ada key berbeda → gagal kecuali `--force`
4. **Validasi ke server** (kecuali `--no-validate`)
5. **Simpan ke encrypted storage** (OS keyring atau encrypted file fallback)
6. **Hapus baris `LICENSE=`** dari `.env` (kecuali `--keep-env`):
   - Atomic write: stage ke temp file, lalu rename (analog `os.replace`)
   - Preserve: line ending (CRLF/LF), comment, blank line, key-value lain
   - Cache `.env` di-invalidate otomatis

## Mengapa Perlu Migrasi

File `.env` masih didukung sebagai source license (priority 3 di chain) tetapi punya beberapa kelemahan:

| Kelemahan `.env` | Solusi `encrypted storage` |
|-------------------|--------------------------|
| Plain text — license terbaca siapapun yang akses file | Disimpan terenkripsi di OS keyring atau encrypted file fallback |
| Per-project — perlu copy ke setiap project | Per-user — sekali setup berlaku semua project |
| Mudah ter-commit ke git tanpa sengaja | Tidak ada file di project root yang berisi key |
| Tidak terikat hardware | Encrypted file fallback terikat ke mesin, tidak portable |

Migrasi diperlukan untuk user yang **upgrade dari workflow lama** ke pola encrypted storage.

## Idempotency

Jika key di `.env` sama dengan key di encrypted storage, command treat sebagai idempotent:

- Tidak save ulang ke storage (no-op)
- Tetap hapus baris dari `.env` (kecuali `--keep-env`)

Hasil: setelah migrate sukses, `.env` bersih dan storage berisi key valid.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Migrasi sukses (atau no-op idempotent) |
| `1` | Tidak ada key di `.env`, format invalid, server validation gagal (non-`--no-validate`), encrypted existing tanpa `--force`, atau IO error |

## Hubungan dengan `activate`

| Aspek | `activate --key=XXX` | `license migrate` |
|-------|---------------------|-------------------|
| Source key | CLI flag atau prompt | File `.env` (`LICENSE=`) |
| Hapus dari `.env` | Tidak | Ya (kecuali `--keep-env`) |
| Use case | Setup awal, key baru dari email/portal | Upgrade dari workflow lama (`.env`-based) |
| Validation server | Default ya | Default ya (skip dengan `--no-validate`) |

## Skenario Umum

| Skenario | Command |
|----------|---------|
| Pindah workflow dari `.env` ke encrypted | `restforge-designer license migrate` |
| Migrasi tetapi simpan `.env` (untuk audit) | `restforge-designer license migrate --keep-env` |
| Migrasi di mesin offline | `restforge-designer license migrate --no-validate` |
| Force overwrite key di storage dengan key di `.env` | `restforge-designer license migrate --force` |

## Verifikasi Setelah Migrasi

```bash
restforge-designer license status
```

Field `Resolved from` harus menampilkan `encrypted storage` (bukan `.env file (legacy)`).

---

**Lihat juga**: [`README`](./README.md) · [`activate.md`](./activate.md) · [`license-status.md`](./license-status.md) · [`deactivate.md`](./deactivate.md)
