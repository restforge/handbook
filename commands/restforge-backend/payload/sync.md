# `payload sync`

> Menerapkan schema drift dari database ke file payload (update kolom, tipe, constraint, dll.).

## Pattern

```
npx restforge payload sync --config=<FILE> [--table=<NAME>]
npx restforge payload sync --table=<NAME> --expand-fk [--fk-columns=<table.column,...>]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak¹ | semua | Sync hanya satu tabel spesifik |
| `--expand-fk` | Tidak | `false` | Generate konfigurasi JOIN dari foreign key (lihat [Ekspansi Foreign Key](#ekspansi-foreign-key---expand-fk)) |
| `--fk-columns <LIST>` | Tidak | auto | Kolom tabel referensi yang di-surface, format `table.column` dipisah koma. Bila dikosongkan, di-resolve otomatis per FK |

¹ `--table` wajib disertakan saat `--expand-fk` aktif (ekspansi bekerja pada satu tabel target).

## Contoh (Examples)

```bat
npx restforge payload sync --config=db.env
```

## Ekspansi Foreign Key (`--expand-fk`)

Flag `--expand-fk` membuat `payload sync` menghasilkan konfigurasi JOIN dari
foreign key tabel target, sehingga kolom display dari tabel referensi (mis.
`category_name`) ikut muncul di endpoint `/datatables`, `/read`, `/first`, dan
`/lookup`. Tanpa flag ini, response endpoint bersifat flat dan hanya memuat kolom
FK (mis. `category_id`).

Flag bersifat opt-in. Tanpa `--expand-fk`, perilaku `payload sync` tidak berubah
sama sekali.

### Yang Dihasilkan (Output)

Saat dijalankan dengan `--expand-fk`, sync melakukan hal berikut pada tabel target:

1. Menulis file SQL JOIN ke `payload/query/<table>-join.sql`.
2. Merevisi file payload:
   - `fieldName` ditambah kolom tabel referensi yang di-surface
   - `datatablesQuery` dan `viewQuery` ditetapkan ke `file:query/<table>-join.sql`
   - `datatablesWhere` ditambah kolom referensi sebelum entri `"all"`
   - `fieldValidation` **tidak disentuh** (kolom JOIN bukan kolom fisik tabel)
3. Mengarsipkan file payload lama menjadi `<file>.json.archive.NNN` sebelum menulis revisi.

Berbeda dengan sync biasa, mode `--expand-fk` tetap memproses tabel meskipun
kolom fisik sudah selaras. Bila terdapat drift kolom fisik, kolom direkonsiliasi
terlebih dahulu, lalu ekspansi FK diterapkan di atasnya.

### Mode Pemilihan Kolom (Column Selection)

Terdapat dua mode penentuan kolom tabel referensi yang di-surface:

**Mode eksplisit** memakai `--fk-columns`. Setiap entri WAJIB berbentuk
`table.column` (qualified), dipisah koma. Bentuk polos tanpa nama tabel ditolak
agar tidak ada penebakan tabel referensi pada kasus multi-FK.

```bat
npx restforge payload sync --table=visitors --expand-fk ^
  --fk-columns=visitor_categories.category_code,visitor_categories.category_name
```

**Mode auto-resolve** aktif bila `--fk-columns` dikosongkan. Untuk SETIAP foreign
key, satu kolom display dipilih dari tabel referensi dengan urutan prioritas:

1. Kolom name: token `name` atau `nama`
2. Kolom code: token `code` atau `kode` (bila tidak ada kolom name)
3. Primary key tabel referensi (fallback terakhir)

Dalam tiap kelompok, exact match menang atas berakhiran `_<token>`, yang menang
atas sekadar mengandung token. Kolom audit di-exclude dari kandidat. Pemilihan
dicetak sebagai baris `[AUTO] <table> -> <ref>.<col> (display column)`.

```bat
:: visitor_categories memiliki category_name -> setara dengan
:: --fk-columns=visitor_categories.category_name
npx restforge payload sync --table=visitors --expand-fk
```

Bila tabel referensi tidak memiliki kolom name/code maupun primary key, sync
gagal dengan pesan eksplisit yang menyarankan penggunaan `--fk-columns`.

### Contoh SQL JOIN (Single FK)

Tabel `visitors` dengan FK `category_id` ke `visitor_categories` menghasilkan
`payload/query/visitors-join.sql`:

```sql
SELECT v.visitor_id,
       v.name,
       v.email,
       v.phone,
       v.category_id,
       vc.category_code,
       vc.category_name
FROM visitors v
LEFT JOIN visitor_categories vc ON vc.category_id = v.category_id
```

Alias tabel diturunkan secara deterministik dari inisial kata yang dipisah
underscore (`visitors` menjadi `v`, `visitor_categories` menjadi `vc`). Seluruh
relasi memakai `LEFT JOIN` agar record tanpa nilai FK tetap muncul dengan kolom
referensi bernilai `NULL`.

### Contoh Multi-FK

Tabel `stock_inbound` dengan dua FK (`warehouse_id` ke `warehouse`, `supplier_id`
ke `supplier`) menghasilkan dua `LEFT JOIN`:

```sql
SELECT si.stock_inbound_id,
       si.inbound_number,
       si.warehouse_id,
       si.supplier_id,
       w.warehouse_name,
       s.supplier_name
FROM stock_inbound si
LEFT JOIN warehouse w ON w.warehouse_id = si.warehouse_id
LEFT JOIN supplier s ON s.supplier_id = si.supplier_id
```

### Penanganan Collision Nama Output

Bila dua kolom yang di-surface bernama sama (mis. `warehouse.city` dan
`supplier.city`), nama output di-prefix nama tabel referensi via alias `AS`:

```sql
       w.city AS warehouse_city,
       s.city AS supplier_city,
```

dan `fieldName` memuat `warehouse_city` serta `supplier_city`. Tanpa collision,
nama kolom dipertahankan apa adanya.

> **Catatan**: Bila `--fk-columns` menyebut tabel yang tidak direferensikan FK
> mana pun, kolom yang tidak ada di tabel referensi, atau terdapat FK ganda ke
> tabel yang sama (tidak dapat di-disambiguasi via nama tabel), sync gagal dengan
> pesan eksplisit tanpa menulis perubahan.

Referensi terkait: [`catalogs/rdf/data-source.md`](../../../catalogs/rdf/data-source.md)
(prioritas `viewName` → `viewQuery` → `tableName`), [`catalogs/rdf/file-reference.md`](../../../catalogs/rdf/file-reference.md)
(referensi SQL via `file:` prefix), dan [`catalogs/rdf/field-lookup.md`](../../../catalogs/rdf/field-lookup.md).

## Resolusi Audit Columns (Audit Columns Resolution)

Command `payload sync` juga memverifikasi alignment antara
konfigurasi `auditColumns` di payload dengan keberadaan kolom audit standar
(`created_at`, `created_by`, `updated_at`, `updated_by`) di tabel database.
Tujuannya menghilangkan inkonsistensi sinyal antara `payload sync` (yang
sebelumnya tidak menyentuh `auditColumns`) dan `endpoint create` (yang sudah
audit-column-aware).

Matriks resolusi:

| Kondisi Payload | Kondisi Database | Aksi Sync |
|----------------|-----------------|-----------|
| `auditColumns` tidak ditetapkan (default true) | 4 kolom audit standar ada | No-op, payload tidak berubah |
| `auditColumns` tidak ditetapkan (default true) | Tidak ada kolom audit | Set `"auditColumns": false`, info log dicetak |
| `auditColumns` tidak ditetapkan (default true) | Partial (1-3 kolom audit) | Set `"auditColumns": false` (shape audit tidak lengkap, diperlakukan sebagai tidak lengkap) |
| `auditColumns: true` (eksplisit) | Tidak ada kolom audit | Override jadi `false`, warning ke stderr |
| `auditColumns: false` / `null` (eksplisit) | Apa pun | No-op, preserve |
| `auditColumns: { ... }` (object form) | Apa pun | Warning, tidak auto-override (manual verification) |

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

> **Catatan**: Pencocokan nama kolom audit dilakukan exact lowercase (snake_case).
> Bila tabel pakai konvensi berbeda (mis. `CreatedAt`), audit detection akan
> miss dan `auditColumns` ditetapkan `false`. Bila memang ingin pakai custom column
> names, gunakan object form `auditColumns` di payload.

## Re-generate Field Validation

`payload sync` me-regenerate `fieldValidation` dari schema database terkini menggunakan rules yang sama dengan [`payload generate`](./generate.md#field-validation-otomatis). Implikasi untuk payload existing yang dihasilkan sebelum dukungan derive constraint string:

- Field string non-PK dengan `NOT NULL` di database yang sebelumnya tidak punya entry di `fieldValidation` akan **ditambahkan** dengan `required: true`.
- Field string non-PK dengan `character_maximum_length` akan mendapat entry `maxLength`.
- Field string non-PK yang memiliki UNIQUE constraint single-column akan mendapat entry `unique: true`.

Review diff hasil sync sebelum commit agar perubahan behavior endpoint `/create` dan `/update` dapat diantisipasi. Customisasi manual pada entry existing (`format`, `pattern`, `minLength`, custom error message, dan constraint lain yang tidak di-derive dari DB) tetap dipertahankan oleh sync.

## Default Scope `is_active` (Built-in)

`payload sync` ikut menjaga `defaultScope` berbasis kolom `is_active` agar selaras dengan struktur tabel terkini, konsisten dengan perilaku [`payload generate`](./generate.md#default-scope-otomatis-is_active).

| Kondisi Tabel | Aksi Sync |
|---------------|-----------|
| Kolom `is_active` ada di tabel (dan `fieldName`) | Pastikan `defaultScope` memuat `is_active: true` pada `lookup` dan `read` (di-merge bila `defaultScope` sudah ada) |
| Kolom `is_active` dihapus dari tabel | Lepas key `is_active` dari `defaultScope.lookup`/`read`; bila scope menjadi kosong, `defaultScope` ikut dihapus |

Sinkronisasi bersifat **presisi**: hanya key `is_active` yang disentuh. Filter scope kustom lain (mis. `tenant_id`, `show_in_store`) tetap dipertahankan. Penghapusan saat kolom hilang berjalan melalui jalur deteksi drift: menghapus kolom `is_active` dari tabel terdeteksi sebagai drift, sehingga sync memproses file (bukan `[SKIP]`) lalu melepas `defaultScope`.

Konsep, perilaku runtime, dan nilai yang didukung dijelaskan di [`catalogs/rdf/default-scope.md`](../../../catalogs/rdf/default-scope.md) dan [`features/default-scope`](../../../features/default-scope/README.md).

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
