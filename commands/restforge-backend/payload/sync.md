# `payload sync`

> Menerapkan schema drift dari database ke file payload (update kolom, tipe, constraint, dll.).

## Pattern

```
npx restforge payload sync --config=<FILE> [--table=<NAME>] [--schema-path=<PATH>]
npx restforge payload sync --table=<NAME> --expand-fk=both|datatables-only [--fk-columns=<SPEC,...>] [--expand-fk-skip=<SPEC,...>] [--no-interactive]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--config <FILE>` | Ya | - | File config database |
| `--table <NAME>` | Tidak¹ | semua | Sync hanya satu tabel spesifik |
| `--schema-path <PATH>` | Tidak | `schema` | Lokasi SDF (file atau folder) untuk menurunkan CHECK dan foreign key. Lihat [Turunan CHECK dan Foreign Key dari SDF](#turunan-check-dan-foreign-key-dari-sdf) |
| `--expand-fk <MODE>` | Tidak | - | Mode ekspansi JOIN: `both` (ubah `datatablesQuery` dan `viewQuery`) atau `datatables-only` (ubah `datatablesQuery` saja, `viewQuery` tidak berubah). Lihat [Ekspansi Foreign Key](#ekspansi-foreign-key---expand-fk) |
| `--fk-columns <LIST>` | Tidak | auto | Kolom tabel referensi yang dimunculkan. Format `ref_table.column` atau `local_col:ref_table.column` untuk FK ganda ke tabel yang sama, dipisah koma. Bila dikosongkan, ditentukan otomatis per FK |
| `--expand-fk-skip <LIST>` | Tidak | - | Relasi yang tidak ikut di-JOIN. Format `ref_table` atau `local_col:ref_table` untuk FK ganda ke tabel yang sama, dipisah koma. FK lain tetap diekspansi. Lihat [Melewati Relasi Tertentu](#melewati-relasi-tertentu) |
| `--no-interactive` | Tidak | interaktif | Jangan bertanya saat kolom tampilan tidak ditemukan; sync langsung gagal dengan petunjuk. Tanpa terminal (MCP, CI, pipe), sync selalu berjalan non-interaktif |

¹ `--table` wajib disertakan saat `--expand-fk` aktif (ekspansi bekerja pada satu tabel target).

## Contoh

```bat
npx restforge payload sync --config=db.env
```

## Ekspansi Foreign Key (`--expand-fk`)

Flag `--expand-fk` membuat `payload sync` menghasilkan konfigurasi JOIN dari
foreign key tabel target, sehingga kolom tampilan dari tabel referensi (mis.
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
   - Kolom referensi yang ditambahkan bersifat **baca dan cari saja**: kolom itu muncul
     di response dan dapat dipakai sebagai target pencarian, tetapi tidak pernah ditulis
     oleh `/create` maupun `/update` karena bukan kolom fisik tabel
3. Mengarsipkan file payload lama dan file query SQL yang berubah ke `.restforge/archive/<YYYYMMDD-HHMMSS>/` sebelum menulis revisi (lihat [Arsip File Lama](../conventions.md#arsip-file-lama)).

Berbeda dengan sync biasa, mode `--expand-fk` tetap memproses tabel meskipun
kolom fisik sudah selaras. Bila terdapat drift kolom fisik, kolom direkonsiliasi
terlebih dahulu, lalu ekspansi FK diterapkan di atasnya.

### Mode Pemilihan Kolom

Terdapat dua mode penentuan kolom tabel referensi yang dimunculkan:

**Mode auto-resolve** aktif bila `--fk-columns` dikosongkan. Untuk setiap foreign
key, satu kolom tampilan dipilih dari tabel referensi dengan urutan prioritas:

1. Kolom name: token `name` atau `nama`
2. Kolom code: token `code` atau `kode`
3. Kolom nomor: token `number`, `nomor`, `no`, atau `num` (mis. `document_number`, `invoice_no`)
4. Kolom judul: token `title`, `judul`, atau `label`
5. Kolom UNIQUE bertipe teks pertama menurut urutan kolom tabel

Dalam tiap kelompok, kecocokan persis menang atas berakhiran `_<token>`. Kelompok
name dan code juga menerima kolom yang sekadar mengandung token. Kelompok nomor dan
judul lebih ketat: token hanya cocok sebagai nama utuh, awalan `<token>_`, atau
akhiran `_<token>`, sehingga `notes` dan `nominal` tidak terpilih sebagai kolom nomor.
Kolom audit dikecualikan dari kandidat, dan primary key tidak pernah dipilih pada
kelompok 3 sampai 5. Pemilihan dicetak sebagai baris
`[AUTO] <table> -> <ref>.<col> (display column)`.

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

Untuk mengganti kolom tampilan FK tunggal tanpa menyentuh yang lain:

```bat
npx restforge payload sync --table=visitors --expand-fk=both ^
  --fk-columns=visitor_categories.category_code
```

### Kolom Tampilan Tidak Ditemukan

Sync tidak memakai primary key sebagai kolom tampilan. Nilai primary key sama
dengan kolom FK yang sudah ada di payload, sehingga kolom JOIN-nya hanya menjadi
duplikat. Bila kelima kelompok di atas tidak menghasilkan kandidat, perilaku sync
bergantung pada cara command dijalankan.

Di terminal interaktif, sync bertanya kolom mana yang dipakai. Jawab dengan nama
kolom, atau `skip` untuk tidak meng-JOIN relasi tersebut. Jawaban yang tidak cocok
dengan kolom mana pun ditanyakan ulang maksimal tiga kali. Setelah menjawab, sync
mencetak flag setara agar pilihan yang sama bisa diulang tanpa pertanyaan:

```
[AUTO]    roster -> employee.full_name (display column)
[ASK]     roster -> shift_pattern: no natural display column found.
          Available columns: start_time, end_time
          Choose display column (or "skip"): start_time
[EXPLICIT] roster -> shift_pattern.start_time
          Tip: next time use --fk-columns=shift_pattern.start_time
```

Pilihan dari pertanyaan ini tidak disimpan di payload. Sync berikutnya akan
bertanya lagi, kecuali flag yang dicetak pada baris `Tip` ditambahkan ke command.

Tanpa terminal interaktif, misalnya saat dijalankan lewat MCP, CI, pipe, atau
dengan `--no-interactive`, sync tidak bertanya. Payload tabel tersebut tidak
diubah, command berakhir dengan exit code 1, dan pesan error menyebut flag yang
perlu ditambahkan:

```
[ERROR]   roster.json - expand-fk failed: No natural display column found for referenced table "shift_pattern". Available columns: start_time, end_time. Re-run with --fk-columns=shift_pattern.<column> or --expand-fk-skip=shift_pattern
```

> **Catatan**: Versi sebelumnya memakai primary key sebagai pilihan terakhir dan
> menghasilkan kolom seperti `overtime_request_overtime_request_id`. Saat payload
> seperti itu di-sync ulang dengan `--expand-fk`, kolom tersebut diganti kolom
> tampilan yang ditemukan (mis. `document_number`). Bila kolom tidak ditemukan,
> sync bertanya atau gagal seperti dijelaskan di atas. Periksa frontend yang masih
> memakai nama kolom lama.

### Melewati Relasi Tertentu

`--expand-fk-skip` mengecualikan relasi dari JOIN tanpa meninggalkan mode
auto-resolve untuk FK lain. Pakai flag ini untuk tabel referensi yang memang tidak
punya kolom yang layak ditampilkan:

```bat
npx restforge payload sync --table=roster --expand-fk=both --expand-fk-skip=shift_pattern
```

```
[SKIP]    roster -> shift_pattern (excluded by --expand-fk-skip)
[AUTO]    roster -> employee.full_name (display column)
```

Untuk FK ganda ke tabel yang sama, format `local_col:ref_table` hanya melewati satu
FK, misalnya `--expand-fk-skip=approver_id:employee`. Sync gagal dengan pesan
eksplisit pada tiga kondisi berikut:

- Entri `--expand-fk-skip` tidak cocok dengan foreign key mana pun.
- Relasi yang sama disebut di `--fk-columns` dan `--expand-fk-skip`.
- Semua foreign key tabel dilewati, sehingga tidak ada yang bisa diekspansi.

### Contoh SQL JOIN (Single FK)

Tabel `visitors` dengan FK `category_id` ke `visitor_categories` menghasilkan
`payload/query/visitors-join.sql`:

```sql
SELECT a.visitor_id,
       a.name,
       a.email,
       a.phone,
       a.category_id,
       b.category_code,
       b.category_name
FROM visitors a
LEFT JOIN visitor_categories b ON b.category_id = a.category_id
```

Alias tabel berupa satu huruf berurutan. Tabel target selalu `a`, lalu tabel
referensi mendapat `b`, `c`, dan seterusnya sesuai urutan relasi. Alias tidak
diturunkan dari nama tabel, sehingga tidak pernah bentrok dengan kata kunci SQL
seperti `or`, `on`, atau `as`. Seluruh relasi memakai `LEFT JOIN` agar record
tanpa nilai FK tetap muncul dengan kolom referensi bernilai `NULL`.

### Contoh Multi-FK

Tabel `stock_inbound` dengan dua FK (`warehouse_id` ke `warehouse`, `supplier_id`
ke `supplier`) menghasilkan dua `LEFT JOIN`:

```sql
SELECT a.stock_inbound_id,
       a.inbound_number,
       a.warehouse_id,
       a.supplier_id,
       b.warehouse_name,
       c.supplier_name
FROM stock_inbound a
LEFT JOIN warehouse b ON b.warehouse_id = a.warehouse_id
LEFT JOIN supplier c ON c.supplier_id = a.supplier_id
```

### Batas Jumlah JOIN

Satu file SQL JOIN memuat paling banyak 25 tabel referensi, yaitu alias `b`
sampai `z`. Beberapa kolom dari tabel referensi yang sama dihitung satu JOIN,
sedangkan FK ganda ke tabel yang sama dihitung per kolom FK karena masing-masing
mendapat JOIN sendiri.

Bila jumlahnya melebihi batas, sync gagal untuk tabel tersebut. File SQL JOIN
tidak ditulis dan payload tidak diubah:

```
[ERROR]   wide-table.json - expand-fk failed: Table "wide_table" requires 26 JOINs for --expand-fk, but the maximum is 25. Split the query into subqueries or a CTE, or exclude relations with --expand-fk-skip
```

Pecah query menjadi subquery atau CTE di file SQL kustom, atau kurangi relasi
dengan `--expand-fk-skip`.

> **Catatan**: Versi sebelumnya menurunkan alias dari inisial nama tabel, misalnya
> `visitor_categories` menjadi `vc`. Saat `payload sync --expand-fk` dijalankan
> ulang, alias di file `<table>-join.sql` yang sudah ada berganti menjadi `a`, `b`,
> `c`, dan seterusnya. Nama kolom output tidak bergantung pada alias, sehingga
> response endpoint tidak berubah.

### Penanganan Collision Nama Output

Bila dua kolom yang dimunculkan bernama sama (mis. `warehouse.city` dan
`supplier.city`), nama output diberi prefiks nama tabel referensi via alias `AS`:

```sql
       b.city AS warehouse_city,
       c.city AS supplier_city,
```

dan `fieldName` memuat `warehouse_city` serta `supplier_city`. Tanpa collision,
nama kolom dipertahankan apa adanya.

Untuk FK ganda ke tabel yang sama (disambiguasi via `local_col:ref_table.column`),
prefix yang dipakai adalah nama kolom FK lokal, bukan nama tabel referensi:

```sql
       b.nama AS t_group_id_nama,
       c.nama AS t_group_id_d1_nama,
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

## Kolom yang Dihapus dari `fieldName`

Sync menambahkan kolom baru dari tabel ke `fieldName`, tetapi tidak mengembalikan kolom yang sengaja dihapus dari `fieldName`, misalnya `password`. Kolom yang dihapus juga tidak ditambahkan ke `datatablesWhere` dan `fieldValidation`. Sync membedakan kedua jenis kolom dari daftar kolom di folder `payload/.meta/`, sama seperti [`payload generate`](./generate.md#kolom-yang-dihapus-dari-fieldname).

Daftar kolom diperbarui setiap kali sync dijalankan, termasuk untuk RDF yang sudah selaras dengan database. Kolom yang di-drop dari tabel ikut keluar dari daftar. RDF berstatus `[ERROR]` tidak diubah, dan daftar kolomnya juga tidak diperbarui.

Pada RDF lama yang belum punya daftar kolom, sync pertama tidak menambahkan kolom apa pun dan menampilkan peringatan yang sama dengan `payload generate`. Setelah sync itu, daftar kolom terbentuk dan kolom baru berikutnya ditambahkan seperti biasa.

## Kolom yang Di-drop dari Tabel

Sync mengeluarkan kolom yang di-drop dari tabel dari `fieldName`, `datatablesWhere`, dan `fieldValidation`, walaupun query SQL masih memilih kolom itu. File `query/<table>-datatables.sql` hasil `payload generate` dibuat ulang dari `fieldName` terbaru:

```
  [ARCHIVE] users.json -> .restforge/archive/20260928-182901/payload/users.json
  [QUERY]   users-datatables.sql written (query/users-datatables.sql)
  [SYNCED]  users.json - 1 payload field(s) missing from database
            Query check failed against the database:
              [!] datatablesQuery: 42703: column a.email does not exist
```

Pesan query pada output di atas berasal dari pemeriksaan sebelum sync. Setelah sync, `payload validate` melaporkan `[OK]`.

File SQL yang sudah diedit manual, misalnya ditambah `ORDER BY`, tidak ditimpa. File SQL JOIN dan SQL inline yang ditulis manual juga tidak diubah. Sync menampilkan peringatan berisi nama file dan kolom yang harus dihapus dari query:

```
Warning: manually edited query/users-datatables.sql was kept and still selects column(s) email that no longer exist in table users. Remove them from the query manually.
```

Untuk file SQL JOIN hasil `--expand-fk`, peringatan menyarankan command `--expand-fk` yang membuat ulang query JOIN. Selama query belum diperbaiki, `payload validate` melaporkan RDF tersebut sebagai `[ERROR]` (lihat [Query SQL yang Gagal](./validate.md#query-sql-yang-gagal)).

## Kolom JOIN pada Sync Biasa

Sync tanpa `--expand-fk` tidak membuang kolom JOIN dari RDF. Kolom seperti `supplier_name` tetap ada di `fieldName` dan `datatablesWhere` selama file SQL yang dirujuk `datatablesQuery`, `viewQuery`, atau `exportQuery` masih memilihnya. Referensi `file:` pada ketiga key itu juga tidak diubah.

Kolom baru dari tabel ditambahkan di akhir `fieldName`. Sync biasa tidak menulis ulang file SQL JOIN, sehingga kolom baru belum ikut dipilih. Sync menampilkan peringatan untuk kondisi ini:

```
Warning: query/good-receipt-join.sql does not select column(s) receiver_name from fieldName. Run 'npx restforge payload sync --table=good_receipt --expand-fk=both' to refresh the JOIN query.
```

Jalankan command pada peringatan itu untuk memperbarui SQL JOIN. Bila kolom tampilan dulu dipilih lewat `--fk-columns`, sertakan nilai yang sama. Perilaku ini sama dengan [`payload generate`](./generate.md#generate-ulang-pada-rdf-yang-sudah-ada) saat dijalankan ulang pada RDF yang sudah ada.

SQL inline pada `datatablesQuery` dan `exportQuery` diperlakukan dengan dua cara:

- Query berbentuk `select <kolom fieldName> from <table>` ditulis ulang dengan kolom terbaru.
- Query yang ditulis manual dipertahankan, misalnya query JOIN atau query yang sengaja tidak memilih kolom password. Kolom JOIN dari query tersebut tetap ada di `fieldName` dan `datatablesWhere`.

Kolom baru yang tidak dipilih SQL inline manual pada `datatablesQuery` tidak ditambahkan ke `datatablesWhere`. Sync menampilkan peringatan yang sama dengan [`payload generate`](./generate.md#kolom-baru-dan-sql-join).

Nilai lain di luar kolom dan query juga tidak diubah sync, termasuk nilai action standar seperti `create: false`.

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
| `auditColumns: { ... }` (object form) | Semua kolom pemetaan ada | Tidak ada perubahan, dipertahankan |
| `auditColumns: { ... }` (object form) | Ada kolom pemetaan yang tidak ditemukan | Dipertahankan, peringatan menyebut kolom yang tidak ditemukan |

Output yang relevan:

- Bila sync menetapkan `auditColumns: false` dari perilaku default:
  ```
  [restforge] Set "auditColumns": false in <file> because audit columns missing in database
  ```
- Bila sync override `auditColumns: true` jadi `false`:
  ```
  [restforge] Resetting auditColumns: true -> false for table "<table>" because audit columns missing in database
  ```
- Bila object form memetakan kolom yang tidak ada di tabel:
  ```
  [restforge] Custom auditColumns object detected for "<table>". Column(s) insert_date not found in database. Verify column names manually.
  ```

Kolom yang dipetakan object form, misalnya `insert_date` dan `insert_by`, tidak ditambahkan ke `fieldName` oleh sync. Kolom itu tetap dipertahankan bila sudah tercantum di `fieldName`.

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

Review diff hasil sync sebelum commit agar perubahan behavior endpoint `/create` dan `/update` dapat diantisipasi. Kustomisasi manual pada entri yang ada (`format`, `pattern`, `minLength`, `hash`, custom error message, dan constraint lain yang tidak diturunkan dari DB) tetap dipertahankan oleh sync. Aturan penggabungannya sama dengan [`payload generate`](./generate.md#generate-ulang-pada-rdf-yang-sudah-ada): constraint turunan database diselaraskan, `maxLength` manual yang lebih kecil dipertahankan, dan tipe manual yang sejenis dengan tipe kolom dipertahankan.

## Turunan CHECK dan Foreign Key dari SDF

Sync membaca SDF dari `--schema-path` dan menurunkan [`checks`](../../../catalogs/sdf/check-constraints.md) ke `fieldValidation` (`enum`, `min`, `max`, `notEqual`) serta ke registry [`checkConstraints`](../../../catalogs/rdf/check-constraints.md). Untuk Oracle dan SQLite, relasi `belongsTo` juga diturunkan ke [`foreignKeyConstraints`](../../../catalogs/rdf/foreign-key-constraints.md).

RDF yang kolomnya sudah selaras dengan database tetap diperbarui bila hasil turunan ini belum ada:

```
  [ARCHIVE] guest.json -> .restforge/archive/20261001-084054/payload/guest.json
  [SYNCED]  guest.json - CHECK constraints derived from SDF
```

Constraint yang sudah ada di `fieldValidation` tidak ditimpa. Bila SDF tidak ditemukan, turunan dilewati dan nilai di RDF dipertahankan.

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
