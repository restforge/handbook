# Workflow Sinkronisasi SDF dengan Database (SDF Database Sync Workflow)

Setelah SDF dideklarasikan, terdapat tiga sub-command CLI yang menangani lifecycle sinkronisasi antara SDF dan struktur tabel actual di database. Ketiganya saling melengkapi, bukan menggantikan satu sama lain.

## Tiga Sub-command, Tiga Use Case Berbeda

| Sub-command | Use Case | Mode Default | Destruktif? |
|---|---|---|---|
| `schema migrate` | Inisialisasi awal: buat tabel dari SDF di database kosong, atau rebuild via DROP+CREATE | `CREATE TABLE IF NOT EXISTS` (no-op untuk tabel existing) atau `DROP + CREATE` (via `--drop=true`) | Hanya bila `--drop=true` |
| `schema diff` | Deteksi drift bidirectional antara SDF dan struktur tabel actual (read-only) | Introspeksi DB + comparison terhadap SDF, output human-readable atau JSON | Tidak (read-only) |
| `schema apply` | Resolve drift incremental via `ALTER TABLE` (komplemen `schema diff`) | Generate `ALTER` statement dari delta diff, default additive-only | Opt-in via `--allow-drop` dan `--allow-modify` |

## Diagram Pipeline Drift Workflow

```
                  ┌──────────────────────────────────┐
                  │   SDF (Schema Definition File)   │
                  └──────────────┬───────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
    ┌──────────────┐    ┌────────────────┐   ┌──────────────────┐
    │   migrate    │    │      diff      │   │      apply       │
    │              │    │                │   │                  │
    │ Bootstrap    │    │ Detect drift   │   │ Resolve drift    │
    │ CREATE/DROP  │    │ (read-only)    │   │ via ALTER        │
    └──────┬───────┘    └────────┬───────┘   └────────┬─────────┘
           │                     │                    │
           │                     ▼                    │
           │             ┌──────────────┐             │
           │             │ Drift report │             │
           │             │ exit 0/1/2   │             │
           │             └──────────────┘             │
           │                                          │
           └────────────────────┬─────────────────────┘
                                ▼
                         ┌──────────────┐
                         │   Database   │
                         └──────────────┘
```

## Kapan Pakai `schema migrate`

- Pertama kali setup database (tabel belum ada sama sekali)
- Database dev atau test yang boleh di-rebuild via `DROP+CREATE`
- **TIDAK** cocok untuk apply incremental drift di tabel yang berisi data:
  - Mode default (`--drop=false`) menghasilkan `CREATE TABLE IF NOT EXISTS` yang menjadi no-op bila tabel sudah ada
  - Mode `--drop=true` destruktif (drop seluruh tabel, data hilang permanen)

## Kapan Pakai `schema diff`

- Verifikasi konsistensi SDF dengan struktur tabel actual di database
- CI/CD gate: pipeline fail bila drift terdeteksi (exit code 1)
- Pre-deployment check di production untuk memastikan migration tidak terlewat
- Aman dijalankan kapan saja (read-only, tidak modifikasi database maupun SDF)

Output `schema diff` bersifat bidirectional, menampilkan tiga kategori delta per tabel:

- **Only in SDF**: field/index/constraint yang ada di SDF tapi tidak di DB
- **Only in DB**: field/index/constraint yang ada di DB tapi tidak di SDF
- **Mismatched**: field yang ada di kedua sisi tapi berbeda (type, length, nullable, default, FK action, dll.)

## Kapan Pakai `schema apply`

- Resolve drift yang dideteksi `schema diff` tanpa kehilangan data
- Apply audit fields baru, index tambahan, atau composite unique constraint ke tabel existing
- Komplemen `schema migrate` yang tidak punya mode incremental

Default behavior bersifat konservatif: hanya **additive-only** (`ADD COLUMN`, `CREATE INDEX`, `ADD UNIQUE CONSTRAINT`, `ADD FOREIGN KEY`) yang ter-emit otomatis. Perubahan destruktif memerlukan opt-in eksplisit:

- `--allow-drop` — izinkan `DROP COLUMN`, `DROP INDEX`, `DROP CONSTRAINT UNIQUE`, `DROP FOREIGN KEY`
- `--allow-modify` — izinkan `ALTER COLUMN length`, `ALTER COLUMN nullable`, dan FK action change (`onDelete`/`onUpdate`)

Perilaku FK secara detail (additive, drop, action change, naming, SQLite) dijelaskan di [`foreign-keys.md`](../foreign-keys.md) section "Behavior pada `schema apply`".

Beberapa kategori perubahan **tidak** di-handle saat ini dan akan ditandai sebagai `deferred` di output:

- `ALTER COLUMN type` (butuh data conversion strategy)
- Primary key changes (butuh rebuild table)
- Default value `ALTER`
- Check constraint `ALTER`
- Foreign key changes pada SQLite (butuh rebuild table)

## Best Practice Pipeline

1. **Selalu pakai `--dry-run` lebih dulu** sebelum apply real ke production:

   ```bat
   npx restforge schema apply --path=./schema --config=db.env --dry-run
   ```

2. **Backup database** sebelum jalankan operasi destruktif (`migrate --drop=true` atau `apply --allow-drop`):

   ```bat
   pg_dump -h localhost -U postgres dbname > backup-YYYY-MM-DD.sql
   ```

3. **Test di staging environment** sebelum apply ke production
4. **Gunakan `schema diff` di CI/CD pipeline** sebagai drift detection gate (exit code 1 menggagalkan pipeline)
5. **Verify dengan `schema diff` ulang setelah apply** untuk memastikan exit code 0

## Exit Code Konvensi (CI/CD Integration)

Konsisten lintas tiga sub-command:

| Exit Code | Arti |
|---|---|
| `0` | Sukses (no drift atau drift applicable ter-apply) |
| `1` | Drift terdeteksi (`diff`) atau drift butuh opt-in user (`apply`) |
| `2` | Error sistem (config invalid, koneksi gagal, path tidak ditemukan, dll.) |

---

**Lihat juga**: [`maintenance/`](./) · [`sdf/`](../) · [`README`](../../../README.md)
