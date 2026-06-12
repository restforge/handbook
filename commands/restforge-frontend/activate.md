# `restforge-designer activate`

> Aktivasi license key. Simpan ke encrypted user storage (global per user, bukan per project).

## Sintaks

```bash
restforge-designer activate [OPTIONS]
```

Tanpa `--key`, command masuk mode **interaktif** dengan prompt input license key.

## Flag

| Flag | Tipe | Default | Wajib | Keterangan |
|------|------|---------|:-----:|-----------|
| `--key <KEY>` | string | — | ✗ (prompt jika kosong) | License key format `XXXX-XXXX-XXXX-XXXX` |
| `--no-validate` | flag | `false` | ✗ | Skip validasi ke server (offline activation) |
| `--force` | flag | `false` | ✗ | Overwrite license existing tanpa konfirmasi |
| `--plugins-dir <PATH>` | string | auto-detect | ✗ | Override folder plugins |

## Contoh Penggunaan

### Aktivasi Standar

```bash
restforge-designer activate --key=ABCD-1234-EFGH-5678
```

Output sukses:

```
╭─ Activation Complete ───────────────────────╮
│ Validation : successful                     │
│ Product    : RESTForge Designer             │
│ Expires    : 2026-12-31                     │
│ Storage    : OS keyring                     │
╰─────────────────────────────────────────────╯
```

### Aktivasi Mode Interaktif

```bash
restforge-designer activate
```

CLI akan meminta input:

```
License key: ABCD-1234-EFGH-5678
[validation spinner...]
[panel success...]
```

Echo input license tidak disembunyikan karena license key bukan secret seperti password.

### Aktivasi Offline (Skip Server Validation)

```bash
restforge-designer activate --key=ABCD-1234-EFGH-5678 --no-validate
```

Pakai jika mesin tidak punya akses internet saat setup. Validasi server akan terjadi nanti saat verb berikutnya dijalankan (mis. `generate` atau `preview`).

### Overwrite License Existing

```bash
restforge-designer activate --key=ZZZZ-9999-YYYY-8888 --force
```

Tanpa `--force`, jika sudah ada license aktif yang berbeda, command tolak dengan pesan:

```
╭─ License Already Active ─────────────────────────────────────────╮
│ An active license is already stored: ABCD-1234-EFGH-5678         │
│ Use --force to overwrite, or run `restforge-designer deactivate` │
│ first.                                                           │
╰──────────────────────────────────────────────────────────────────╯
```

## Behavior

Verb `activate`:

1. **Resolve key** dari `--key` atau prompt interaktif. Normalize: `trim()` + `toUpperCase()`
2. **Validasi format** pattern `XXXX-XXXX-XXXX-XXXX` (uppercase A-Z atau 0-9)
3. **Cek existing**: jika sudah ada license berbeda dan tidak ada `--force`, gagal
4. **Validasi ke server** license resmi (kecuali `--no-validate`)
5. **Simpan ke encrypted storage** (backend priority):
   - **Primary**: OS keyring (Windows Credential Manager / macOS Keychain / Linux Secret Service)
   - **Fallback**: encrypted file di folder user config (`~/.restforge-designer/`) bila keyring tidak tersedia
6. **Tampilkan ringkasan** product, expiry, storage method

## Lokasi Storage

Encrypted storage **global per user account**, bukan per project:

| OS | Lokasi |
|----|--------|
| Windows | `C:\Users\<user>\.restforge-designer\` (Windows Credential Manager untuk keyring) |
| macOS | `~/.restforge-designer/` (macOS Keychain) |
| Linux | `~/.restforge-designer/` (Secret Service) |

**Konsekuensi:** sekali `activate` berhasil, license berlaku untuk semua project di mesin tersebut. Tidak perlu re-activate di setiap folder project.

Encrypted file fallback terikat ke mesin tempat aktivasi dilakukan, sehingga tidak portable lintas mesin. Untuk pindah mesin, jalankan `deactivate` di mesin lama lalu `activate` di mesin baru.

## Validasi Format

| Valid | Invalid | Alasan |
|-------|---------|--------|
| `ABCD-1234-EFGH-5678` | `abcd-1234-efgh-5678` | Lowercase auto-normalize ke uppercase |
| `0000-1111-2222-3333` | `ABCD12345EFGH5678` | Harus ada dash separator |
| `XXXX-YYYY-ZZZZ-WWWW` | `ABC-1234-EFGH-5678` | Segmen pertama hanya 3 char |

CLI otomatis melakukan `.trim().toUpperCase()` sebelum validasi.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Aktivasi sukses |
| `1` | Format key invalid, server validation gagal (non-`--no-validate`), license existing berbeda tanpa `--force`, atau storage tidak tersedia |

## Skenario Umum

| Skenario | Command |
|----------|---------|
| Setup awal mesin baru | `restforge-designer activate --key=XXXX-XXXX-XXXX-XXXX` |
| Mesin air-gapped (offline) | `restforge-designer activate --key=XXXX-XXXX-XXXX-XXXX --no-validate` |
| Renewal license (key baru) | `restforge-designer activate --key=NEW-1234-EFGH-5678 --force` |
| Pindah dari `.env` ke encrypted | Lihat [`license-migrate.md`](./license-migrate.md) |

## Verifikasi Setelah Aktivasi

```bash
restforge-designer license status
```

Output yang diharapkan:

```
┌─────────────────────┬───────────────────────────────┐
│ Field               │ Value                         │
├─────────────────────┼───────────────────────────────┤
│ Active key          │ ABCD-1234-EFGH-5678           │
│ Resolved from       │ encrypted storage             │
│ Encrypted storage   │ OS keyring                    │
│ OS keyring          │ yes                           │
└─────────────────────┴───────────────────────────────┘

┌─ Validation Result ─────────────────────────┐
│ Validation : valid                          │
│ Product    : RESTForge Designer             │
│ Expires    : 2026-12-31                     │
└─────────────────────────────────────────────┘
```

---

**Lihat juga**: [`README`](./README.md) · [`deactivate.md`](./deactivate.md) · [`license-status.md`](./license-status.md) · [`license-migrate.md`](./license-migrate.md)
