# `payload sync`

> Menerapkan schema drift dari database ke file payload (update kolom, tipe, constraint, dll.).

## Pattern

```
npx restforge payload sync --config=<FILE> [--table=<NAME>]
npx restforge payload sync --table=<NAME> --expand-fk=both|datatables-only [--fk-columns=<SPEC,...>]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak¹ | semua | Sync hanya satu tabel spesifik |
| `--expand-fk <MODE>` | Tidak | - | Mode ekspansi JOIN: `both` (ubah `datatablesQuery` dan `viewQuery`) atau `datatables-only` (ubah `datatablesQuery` saja, `viewQuery` tidak berubah). Lihat [Ekspansi Foreign Key](#ekspansi-foreign-key---expand-fk) |
| `--fk-columns <LIST>` | Tidak | auto | Kolom tabel referensi yang dimunculkan. Format `ref_table.column` atau `local_col:ref_table.column` untuk FK ganda ke tabel yang sama, dipisah koma. Bila dikosongkan, ditentukan otomatis per FK |

¹ `--table` wajib disertakan saat `--expand-fk` aktif (ekspansi bekerja pada satu tabel target).

## Contoh

```bat
npx restforge payload sync --config=db.env
```

## Ekspansi Foreign Key (`--expand-fk`)

Flag `--expand-fk` membuat `payload sync` menghasilkan konfigurasi JOIN dari
foreign key tabel target, sehingga kolom display dari tabel referensi (mis.
`category_name`) ikut muncul di endpoint `/datatables`, `/read`, `/first`, dan
`/lookup`. Tanpa flag ini, response endpoint bersifat flat dan hanya memuat kolom
FK (mis. `category_id`).

Flag ini tidak aktif secara default. Tanpa `--expand-fk`, perilaku `payload sync` tidak berubah
sama sekali. Dua nilai yang didukung:

- `both` — menulis file SQL JOIN dan memperbarui `datatablesQuery` serta `viewQuery`.
- `datatables-only` — menulis file SQL JOIN dan memperbarui `datatablesQuery` saja;
  `viewQuery` dipertahankan. Gunakan mode ini bila `viewQuery` sudah merujuk file
  SQL kustom yang tidak boleh ditimpa.

### Yang Dihasilkan

Saat dijalankan dengan `--expand-fk`, sync melakukan hal berikut pada tabel target:

1. Menulis file SQL JOIN ke `payload/query/<table>-join.sql`.
2. Merevisi file payload:
   - `fieldName` ditambah kolom tabel referensi yang dimunculkan
   - `datatablesQuery` ditetapkan ke `file:query/<table>-join.sql`
   - `viewQuery` ditetapkan ke `file:query/<table>-join.sql` (hanya mode `both`;
     mode `datatables-only` mempertahankan nilai `viewQuery` yang ada)
   - `datatablesWhere` ditambah kolom referensi sebelum entri `"all"`
   - `fieldValidation` **tidak disentuh** (kolom JOIN bukan kolom fisik tabel)
3. Mengarsipkan file payload lama menjadi `<file>.json.archive.NNN` sebelum menulis revisi.

Berbeda dengan sync biasa, mode `--expand-fk` tetap memproses tabel meskipun
kolom fisik sudah selaras. Bila terdapat drift kolom fisik, kolom direkonsiliasi
terlebih dahulu, lalu ekspansi FK diterapkan di atasnya.

### Mode Pemilihan Kolom

Terdapat dua mode penentuan kolom tabel referensi yang dimunculkan:

**Mode auto-resolve** aktif bila `--fk-columns` dikosongkan. Untuk setiap foreign
key, satu kolom display dipilih dari tabel referensi dengan urutan prioritas:

1. Kolom name: token `name` atau `nama`
2. Kolom code: token `code` atau `kode` (bila tidak ada kolom name)
3. Primary key tabel referensi (fallback terakhir)

Dalam tiap kelompok, kecocokan persis menang atas berakhiran `_<token>`, yang menang
atas sekadar mengandung token. Kolom audit dikecualikan dari kandidat. Pemilihan
dicetak sebagai baris `[AUTO] <table> -> <ref>.<col> (display column)`.

```bat
:: semua FK di-resolve otomatis, termasuk FK ganda ke tabel yang sama
npx restforge payload sync --table=t_asset --expand-fk=both
```

Bila tabel memiliki **dua FK ke tabel referensi yang sama** (mis. `t_group_id` dan
`t_group_id_d1` keduanya ke `t_group`), auto-resolve secara otomatis memberi prefix
nama kolom FK lokal pada nama output:

```
[AUTO]    t_asset.t_group_id    -> t_group.nama  (display column, auto-disambiguated)
[AUTO]    t_asset.t_group_id_d1 -> t_group.nama  (display column, auto-disambiguated)
[AUTO]    t_asset -> t_location.nama              (display column)
```

Nama kolom output yang dihasilkan: `t_group_id_nama` dan `t_group_id_d1_nama`.

Bila tabel referensi tidak memiliki kolom name/code maupun primary key, sync
gagal dengan pesan eksplisit yang menyarankan penggunaan `--fk-columns`.

**Mode eksplisit (hybrid)** aktif bila `--fk-columns` diisi. `--fk-columns` berlaku
sebagai **override** untuk FK yang disebut; FK lain yang tidak disebut tetap
di-auto-resolve. Format entri:

- `ref_table.column` — untuk FK tunggal ke tabel referensi
- `local_fk_col:ref_table.column` — untuk override FK tertentu saat ada FK ganda

```bat
:: hanya t_group_id dan t_group_id_d1 di-override; t_location, t_satuan, dst. auto
npx restforge payload sync --table=t_asset --expand-fk=both ^
  --fk-columns=t_group_id:t_group.nama,t_group_id_d1:t_group.masa_manfaat
```

```
[EXPLICIT] t_asset.t_group_id    -> t_group.nama
[EXPLICIT] t_asset.t_group_id_d1 -> t_group.masa_manfaat
[AUTO]     t_asset -> t_location.nama              (display column)
[AUTO]     t_asset -> t_satuan.nama                (display column)
```

Untuk mengganti display column FK tunggal tanpa menyentuh yang lain:

```bat
npx restforge payload sync --table=visitors --expand-fk=both ^
  --fk-columns=visitor_categories.category_code
```

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

Bila dua kolom yang dimunculkan bernama sama (mis. `warehouse.city` dan
`supplier.city`), nama output diberi prefiks nama tabel referensi via alias `AS`:

```sql
       w.city AS warehouse_city,
       s.city AS supplier_city,
```

dan `fieldName` memuat `warehouse_city` serta `supplier_city`. Tanpa collision,
nama kolom dipertahankan apa adanya.

Untuk FK ganda ke tabel yang sama (disambiguasi via `local_col:ref_table.column`),
prefix yang dipakai adalah nama kolom FK lokal, bukan nama tabel referensi:

```sql
       g1.nama AS t_group_id_nama,
       g2.nama AS t_group_id_d1_nama,
```

> **Catatan**: Bila `--fk-columns` menyebut tabel yang tidak direferensikan FK
> mana pun, atau kolom yang tidak ada di tabel referensi, sync gagal dengan pesan
> eksplisit tanpa menulis perubahan. FK ganda ke tabel yang sama ditangani
> otomatis di mode auto-resolve. Di mode eksplisit, gunakan format
> `local_fk_col:ref_table.column` untuk mengarahkan override ke FK yang tepat;
> jika hanya menyebut nama tabel tanpa prefix FK lokal saat ada FK ganda, sync
> gagal dengan pesan eksplisit.

Referensi terkait: [`catalogs/rdf/data-source.md`](../../../catalogs/rdf/data-source.md)
(prioritas `viewName` → `viewQuery` → `tableName`), [`catalogs/rdf/file-reference.md`](../../../catalogs/rdf/file-reference.md)
(referensi SQL via `file:` prefix), dan [`catalogs/rdf/field-lookup.md`](../../../catalogs/rdf/field-lookup.md).

## Resolusi Audit Columns

Command `payload sync` juga memverifikasi alignment antara
konfigurasi `auditColumns` di payload dengan keberadaan kolom audit standar
(`created_at`, `created_by`, `updated_at`, `updated_by`) di tabel database.
Tujuannya menghilangkan inkonsistensi sinyal antara `payload sync` (yang
sebelumnya tidak menyentuh `auditColumns`) dan `endpoint create` (yang sudah
audit-column-aware).

Matriks resolusi:

| Kondisi Payload | Kondisi Database | Aksi Sync |
|----------------|-----------------|-----------|
| `auditColumns` tidak ditetapkan (default true) | 4 kolom audit standar ada | Tidak ada perubahan, payload tidak berubah |
| `auditColumns` tidak ditetapkan (default true) | Tidak ada kolom audit | Set `"auditColumns": false`, info log dicetak |
| `auditColumns` tidak ditetapkan (default true) | Partial (1-3 kolom audit) | Set `"auditColumns": false` (shape audit tidak lengkap, diperlakukan sebagai tidak lengkap) |
| `auditColumns: true` (eksplisit) | Tidak ada kolom audit | Ditimpa jadi `false`, peringatan ke stderr |
| `auditColumns: false` / `null` (eksplisit) | Apa pun | Tidak ada perubahan, dipertahankan |
| `auditColumns: { ... }` (object form) | Apa pun | Peringatan, tidak ditimpa otomatis (verifikasi manual) |

Output yang relevan:

- Bila sync menetapkan `auditColumns: false` dari perilaku default:
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
edit secara manual file payload.

> **Catatan**: Pencocokan nama kolom audit dilakukan exact lowercase (snake_case).
> Bila tabel pakai konvensi berbeda (mis. `CreatedAt`), audit detection akan
> miss dan `auditColumns` ditetapkan `false`. Bila memang ingin pakai custom column
> names, gunakan object form `auditColumns` di payload.

## Re-generate Field Validation

`payload sync` membuat ulang `fieldValidation` dari schema database terkini menggunakan rules yang sama dengan [`payload generate`](./generate.md#field-validation-otomatis). Implikasi untuk payload existing yang dihasilkan sebelum dukungan derive constraint string:

- Field string non-PK dengan `NOT NULL` di database yang sebelumnya tidak punya entry di `fieldValidation` akan **ditambahkan** dengan `required: true`.
- Field string non-PK dengan `character_maximum_length` akan mendapat entry `maxLength`.
- Field string non-PK yang memiliki UNIQUE constraint single-column akan mendapat entry `unique: true`.

Review diff hasil sync sebelum commit agar perubahan behavior endpoint `/create` dan `/update` dapat diantisipasi. Kustomisasi manual pada entry existing (`format`, `pattern`, `minLength`, custom error message, dan constraint lain yang tidak diturunkan dari DB) tetap dipertahankan oleh sync.

## Default Scope `is_active` (Built-in)

`payload sync` ikut menjaga `defaultScope` berbasis kolom `is_active` agar selaras dengan struktur tabel terkini, konsisten dengan perilaku [`payload generate`](./generate.md#default-scope-otomatis-is_active).

| Kondisi Tabel | Aksi Sync |
|---------------|-----------|
| Kolom `is_active` ada di tabel (dan `fieldName`) | Pastikan `defaultScope` memuat `is_active: true` pada `lookup` dan `read` (digabung bila `defaultScope` sudah ada) |
| Kolom `is_active` dihapus dari tabel | Lepas key `is_active` dari `defaultScope.lookup`/`read`; bila scope menjadi kosong, `defaultScope` ikut dihapus |

Sinkronisasi bersifat **presisi**: hanya key `is_active` yang disentuh. Filter scope kustom lain (mis. `tenant_id`, `show_in_store`) tetap dipertahankan. Penghapusan saat kolom hilang berjalan melalui jalur deteksi drift: menghapus kolom `is_active` dari tabel terdeteksi sebagai drift, sehingga sync memproses file (bukan `[SKIP]`) lalu melepas `defaultScope`.

Konsep, perilaku runtime, dan nilai yang didukung dijelaskan di [`catalogs/rdf/default-scope.md`](../../../catalogs/rdf/default-scope.md) dan [`features/default-scope`](../../../features/default-scope/README.md).

---

**Lihat juga**: [`payload/`](./) · [`commands/`](../) · [`README`](../../../README.md)
