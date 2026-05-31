# `payload sync`

> Menerapkan schema drift dari database ke file payload (update kolom, tipe, constraint, dll.).

## Pattern

```
npx restforge payload sync --config=<FILE> [--table=<NAME>]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak | semua | Sync hanya satu tabel spesifik |

## Contoh (Examples)

```bat
npx restforge payload sync --config=db.env
```

## Resolusi Audit Columns (Audit Columns Resolution)

Sejak versi 4.5.0, command `payload sync` juga memverifikasi alignment antara
konfigurasi `auditColumns` di payload dengan keberadaan kolom audit standar
(`created_at`, `created_by`, `updated_at`, `updated_by`) di tabel database.
Tujuannya menghilangkan inkonsistensi sinyal antara `payload sync` (yang
sebelumnya tidak menyentuh `auditColumns`) dan `endpoint create` (yang sudah
audit-column-aware sejak versi 4.3.0).

Matriks resolusi:

| Kondisi Payload | Kondisi Database | Aksi Sync |
|----------------|-----------------|-----------|
| `auditColumns` tidak di-set (default true) | 4 kolom audit standar ada | No-op, payload tidak berubah |
| `auditColumns` tidak di-set (default true) | Tidak ada kolom audit | Set `"auditColumns": false`, info log di-print |
| `auditColumns` tidak di-set (default true) | Partial (1-3 kolom audit) | Set `"auditColumns": false` (shape audit tidak lengkap, treat as incomplete) |
| `auditColumns: true` (eksplisit) | Tidak ada kolom audit | Override jadi `false`, warning ke stderr |
| `auditColumns: false` / `null` (eksplisit) | Apapun | No-op, preserve |
| `auditColumns: { ... }` (object form) | Apapun | Warning, tidak auto-override (manual verification) |

Output yang relevan:

- Bila sync set `auditColumns: false` dari default behavior:
  ```
  [restforge] Set "auditColumns": false in <file> because audit columns missing in database
  ```
- Bila sync override `auditColumns: true` jadi `false`:
  ```
  [restforge] Resetting auditColumns: true -> false for table "<table>" because audit columns missing in database
  ```
- Bila payload pakai object form (custom mapping):
  ```
  [restforge] Custom auditColumns object detected for "<table>". Cannot auto-resolve. Verify column names manually.
  ```

Setelah resolusi, `endpoint create` lulus validasi schema tanpa user perlu
edit manual file payload.

> **Catatan**: Match nama kolom audit dilakukan exact lowercase (snake_case).
> Bila tabel pakai konvensi berbeda (mis. `CreatedAt`), audit detection akan
> miss dan `auditColumns` di-set `false`. Bila memang ingin pakai custom column
> names, gunakan object form `auditColumns` di payload.

## Re-generate Field Validation

`payload sync` me-regenerate `fieldValidation` dari schema database terkini menggunakan rules yang sama dengan [`payload generate`](./generate.md#field-validation-otomatis). Implikasi untuk payload existing yang dihasilkan sebelum dukungan derive constraint string:

- Field string non-PK dengan `NOT NULL` di database yang sebelumnya tidak punya entry di `fieldValidation` akan **ditambahkan** dengan `required: true`.
- Field string non-PK dengan `character_maximum_length` akan mendapat entry `maxLength`.
- Field string non-PK yang memiliki UNIQUE constraint single-column akan mendapat entry `unique: true`.

Review diff hasil sync sebelum commit agar perubahan behavior endpoint `/create` dan `/update` dapat diantisipasi. Customisasi manual pada entry existing (`format`, `pattern`, `minLength`, custom error message, dan constraint lain yang tidak di-derive dari DB) tetap dipertahankan oleh sync.

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
