# `restforge-designer deactivate`

> Hapus license key dari encrypted storage dan invalidate validation cache.

## Sintaks

```bash
restforge-designer deactivate [OPTIONS]
```

Tanpa `--yes`, command meminta konfirmasi interaktif sebelum menghapus.

## Flag

| Flag | Tipe | Default | Keterangan |
|------|------|---------|-----------|
| `-y`, `--yes` | flag | `false` | Skip konfirmasi prompt |
| `--plugins-dir <PATH>` | string | auto-detect | Override folder plugins |
| `--license <KEY>` | string | dari storage | License key override (jarang dipakai, hanya untuk validasi) |

## Contoh Penggunaan

### Deactivate dengan Konfirmasi

```bash
restforge-designer deactivate
```

CLI menampilkan prompt:

```
This will remove the activated license and clear the validation cache.
Continue? [y/N]: y

╭─ Deactivation Complete ─────────────────────────────╮
│ License removed from encrypted storage              │
│ Validation cache cleared                            │
╰─────────────────────────────────────────────────────╯
```

### Deactivate Skip Konfirmasi (Otomasi Script)

```bash
restforge-designer deactivate --yes
```

Berguna di CI/CD atau script automation yang tidak punya stdin interaktif.

## Behavior

Verb `deactivate`:

1. **Tampilkan konfirmasi** prompt (skip jika `--yes`)
2. **Hapus license key** dari encrypted storage:
   - OS keyring entry (jika ada)
   - Encrypted file fallback di folder user config (jika ada)
3. **Clear validation cache** (cached validation response + metadata terkait)
4. **Tampilkan ringkasan** hasil

## Apa yang TIDAK Dihapus

| Item | Status |
|------|--------|
| License di file `.env` (root project) | Tidak disentuh |
| OS env var `RESTFORGE_DESIGNER_LICENSE_KEY` | Tidak disentuh |
| File `.restforge-designer/` folder itu sendiri | Tetap ada (folder tidak dihapus) |
| Generated apps di folder lain | Tidak disentuh |

Untuk menghapus license dari `.env`, edit file manual atau jalankan urutan: edit `.env` → hapus baris `LICENSE=...`.

## Verifikasi Setelah Deactivate

```bash
restforge-designer license status
```

Output yang diharapkan:

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

Jika `Active key` masih menampilkan value, kemungkinan key masih ada di `.env` atau env var. Cek priority 3 dan 4 di resolusi (lihat [`conventions.md`](./conventions.md)).

## Re-Activate

Setelah deactivate, gunakan `activate` untuk pasang license baru:

```bash
restforge-designer activate --key=NEW-1234-EFGH-5678
```

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Deactivation sukses, atau user menolak konfirmasi (exit graceful) |
| `1` | IO error saat menghapus storage atau cache |

## Skenario Umum

| Skenario | Command |
|----------|---------|
| Pindah ke mesin lain (clear license di mesin lama) | `restforge-designer deactivate --yes` |
| Reset state license untuk debug | `restforge-designer deactivate --yes && restforge-designer activate --key=...` |
| Ganti license key | `restforge-designer activate --key=NEW-... --force` (langsung overwrite, tidak perlu deactivate dulu) |
| Cleanup sebelum uninstall | `restforge-designer deactivate --yes` |

---

**Lihat juga**: [`README`](./README.md) · [`activate.md`](./activate.md) · [`license-status.md`](./license-status.md)
