# `data push`

> Muat isi file envelope JSON (`data-storage/<table>.json`) ke tabel tujuan via INSERT
> batch berdasarkan metadata SDF. Bersifat **append-only**. Mendukung satu tabel
> (`--table`), per schema (`--schema`), atau seluruh tabel terdaftar (`--all-schemas`).

Nama file diturunkan dari SDF dengan aturan **identik** `data pull`, sehingga file hasil
pull dapat langsung di-push tanpa penggantian nama. Metadata kolom dan tipe canonical bersumber
murni dari SDF; konversi nilai per dialect tujuan ditangani otomatis. Penjelasan format
file, resolusi config, dan penanganan lintas-dialect ada di [`README.md`](./README.md).

## Pattern

```
npx restforge data push --table=<NAME> [options]
npx restforge data push --schema=<LIST> [options]
npx restforge data push --all-schemas [options]
```

## Flag

| Flag | Tipe | Default | Wajib | Keterangan |
|------|------|---------|:-----:|-----------|
| `--table=<NAME>` | string | — | ✗* | Nama tabel tujuan (harus terdaftar di SDF). Bentuk `schema.table` untuk tabel ber-schema. Mutual-exclusive dengan `--schema` / `--all-schemas` |
| `--schema=<LIST>` | string | — | ✗* | Filter schema: satu nama atau comma-separated (mis. `public,sales`). Mem-push tabel SDF pada schema tersebut yang punya file di `--storage-path`, urut topological FK parent→child. Mutual-exclusive dengan `--table` / `--all-schemas` |
| `--all-schemas` | flag | `false` | ✗* | Push seluruh tabel SDF yang punya file di `--storage-path` lintas semua schema, urut topological FK parent→child. Mutual-exclusive dengan `--table` / `--schema` |
| `--config=<FILE>` | path | default `.restforge/defaults.json` | ✗ | File config database tujuan (`.env`). Wajib bila tidak ada default |
| `--schema-path=<PATH>` | path | `schema` | ✗ | Lokasi SDF (file atau folder) sumber metadata tabel |
| `--storage-path=<DIR>` | path | `data-storage` | ✗ | Folder sumber file envelope relatif terhadap working directory |
| `--batch-size=<N>` | number | `1000` | ✗ | Ukuran batch INSERT, sekaligus unit commit per batch |
| `--json` | flag | `false` | ✗ | Output ringkasan dalam format JSON (untuk konsumsi CI/CD) |

\* Tepat satu dari `--table` / `--schema` / `--all-schemas` wajib disertakan.

> **Mutual-exclusive**: tepat satu dari `--table`, `--schema`, atau `--all-schemas` wajib
> disertakan. Menyebut lebih dari satu, atau tidak menyebut satupun, ditolak sebagai usage
> error (exit `2`) sebelum koneksi database dibuka.

> `data push` bersifat **append-only**. Mode `upsert`/`replace` belum tersedia pada versi
> ini, sehingga flag `--mode` dan `--force` tidak dikenal dan akan ditolak arg-parser
> (exit `2`).

## Behavior

1. **Resolve config** database tujuan (lihat [README — Resolusi Config](./README.md#resolusi-config-config-resolution)).
2. **Validasi mutual-exclusive**: tepat satu dari `--table` / `--schema` / `--all-schemas`;
   bila lebih dari satu atau tidak ada satupun disebut → exit `2`, sebelum koneksi database.
3. **Load SDF + table-guard** sebelum koneksi database. Pada `--table`, tabel tidak
   terdaftar / ambigu → ditolak (exit `2`). Pada `--schema`, bila tidak ada satupun tabel
   SDF yang cocok dengan schema yang diminta (0-match) → ditolak (exit `2`). Pada
   `--all-schemas`, SDF tanpa tabel sama sekali menghasilkan 0 tabel dan exit `0` (benign).

Langkah berikut dijalankan **per tabel** (sekali pada `--table`, untuk tiap tabel hasil scope pada `--schema` / `--all-schemas`):

4. **Derive nama file** dari SDF (aturan identik pull): tabel ber-schema →
   `{storage-path}/{schema}/{table}.json` (bersarang), tabel tanpa schema →
   `{storage-path}/{table}.json` (flat). Path diturunkan murni dari schema SDF, bukan dari
   field `schema` di envelope. Pada `--table`, file tidak ada → exit `1`. Isi file diurai
   dan divalidasi shape-nya (envelope invalid → exit `1`).
5. **Column match vs SDF** (bukan introspeksi database): setiap kolom envelope wajib
   terdeklarasi di SDF dengan tipe konsisten. Kolom SDF yang tidak muncul di envelope
   diperbolehkan (memakai default/NULL database saat INSERT). Mismatch → exit `1`.
6. **Connect** + validasi identifier tabel/kolom (anti-injection).
7. **INSERT batch + commit per batch**: tiap nilai di-decode per dialect tujuan, INSERT
   parameterized per baris dijalankan per batch dalam satu transaksi (commit per batch).
   INSERT per-baris (bukan multi-row `VALUES`) menjaga paritas lintas-dialect karena
   Oracle tidak mendukung `VALUES (..),(..)` multi-row.
8. **Disconnect** + tampilkan ringkasan.

### Pemeriksaan Schema dan Back-Compat (Schema Cross-Check)

Path file dan namespace tabel diturunkan murni dari SDF, bukan dari envelope. Bila field
`envelope.schema` ada dan **berbeda** dari schema SDF tabel, push tetap berjalan namun
mencetak **warning** ke stderr (informatif, bukan error). Perlakuan ini konsisten dengan
field `envelope.table` yang juga tidak menggantikan metadata SDF.

Untuk back-compat, push membaca envelope `1.0` (tanpa field `schema`) maupun `1.1`. Envelope
`1.0` diperlakukan seakan `schema` bernilai `null`, sehingga tidak memicu warning pada tabel
SDF tanpa schema.

### Append-only dan Konflik Primary Key

Push selalu menambahkan baris (INSERT). Bila baris dengan primary key yang sama sudah
ada di tabel tujuan, database menolak dengan error dan push berhenti (exit `3`). Tidak
ada update otomatis pada baris yang sudah ada.

### Mode Multi-Tabel (`--schema` / `--all-schemas`)

`--schema=<LIST>` mem-push tabel SDF pada schema yang diminta, sedangkan `--all-schemas`
mem-push **setiap tabel** SDF yang memiliki file envelope di `--storage-path` lintas semua
schema. Karakteristiknya:

- **Urutan topological FK (parent→child)**: urutan push diturunkan dari relasi FK di SDF,
  sehingga tabel parent (yang direferensikan) di-push sebelum tabel child. Hal ini
  mencegah INSERT child melanggar FK constraint. Bila SDF mengandung **siklus FK**,
  runtime fallback ke urutan deklarasi SDF dan mencetak warning ke stderr.
- **Topo-sort pada subset `--schema`**: pada mode `--schema`, urutan FK dihitung hanya atas
  **subset** tabel yang masuk scope, bukan seluruh SDF (lihat catatan FK lintas-schema di bawah).
- **Skip-missing**: tabel SDF yang tidak punya file di `--storage-path` **dilewati** (bukan error),
  dicetak sebagai warning ke stderr dan dicatat di array `skipped[]`.
- **Fail-fast**: bila satu tabel gagal (mis. database error), proses berhenti pada tabel
  tersebut. Tidak ada rollback lintas-tabel (lihat [Risiko Partial Commit](#risiko-partial-commit)).

Mode human mencetak satu baris per tabel diakhiri ringkasan agregat, sedangkan mode
`--json` mengeluarkan satu object agregat. Warning skip dan siklus selalu ke stderr
sehingga tidak mencemari stdout `--json`.

#### Catatan FK + Skip-Missing

Bila tabel **parent** dilewati (tidak ada file di `--storage-path`) sementara tabel **child**
memiliki file dengan FK terisi, INSERT child dapat melanggar FK constraint di database
tujuan. Karena parent dilewati diam-diam namun child tetap diproses, pelanggaran ini
muncul sebagai **database error (exit `3`)** pada langkah child, bukan skip diam-diam.
Sesuai sifat fail-fast, tabel/batch yang sudah ter-commit sebelum error tetap tersimpan
(lihat [Risiko Partial Commit](#risiko-partial-commit)).

#### Catatan FK Lintas-Schema pada `--schema` Subset

Karena topo-sort `--schema` hanya mencakup subset tabel pada schema yang diminta, tabel
**parent** dapat berada di schema lain **di luar subset** sehingga tidak ikut di-push. Bila
tabel **child** dalam subset memiliki FK ke parent di luar subset yang belum ter-INSERT,
INSERT child dapat melanggar FK constraint di database tujuan. Pelanggaran ini muncul
sebagai **database error (exit `3`)** pada langkah child tanpa pre-warning, analog dengan
catatan FK + skip-missing di atas.

### Risiko Partial Commit

Karena commit dilakukan per batch, kegagalan pada satu batch me-rollback batch tersebut,
namun batch sebelumnya yang sudah ter-commit tetap tersimpan. Pada kegagalan di tengah,
tabel tujuan dapat berisi sebagian baris. Field `committed_batches` pada output `--json`
membantu mengidentifikasi sejauh mana push berhasil.

Pada mode multi-tabel (`--schema` / `--all-schemas`), sifat ini meluas lintas-tabel karena
tidak ada transaksi tunggal yang membungkus seluruh tabel. Sesuai fail-fast, tabel yang
sudah selesai di-push sebelum tabel yang gagal tetap ter-commit di database tujuan, tanpa
rollback lintas-tabel.

### Output

Mode human (default), `--table`:

```
Pushed 1280 rows to visitors (dialect: postgresql, batches: 2)
```

Mode `--json`, `--table`:

```json
{ "command": "push", "schema": null, "table": "visitors", "rows": 1280, "target_dialect": "postgresql", "committed_batches": 2 }
```

Mode human, `--all-schemas`:

```
Pushed 2 rows to visitor_categories (dialect: postgresql, batches: 1)
Pushed 3 rows to visitors (dialect: postgresql, batches: 1)
Pushed 12 rows to sales.order_item (dialect: postgresql, batches: 1)
Done: 3 tables, 17 rows total (skipped: 0)
```

Mode `--json`, `--all-schemas` (agregat):

```json
{ "command": "push", "scope": "all-schemas", "table_count": 3, "total_rows": 17,
  "tables": [
    { "schema": null,    "table": "visitor_categories", "rows": 2,  "committed_batches": 1 },
    { "schema": null,    "table": "visitors",            "rows": 3,  "committed_batches": 1 },
    { "schema": "sales", "table": "order_item",          "rows": 12, "committed_batches": 1 }
  ],
  "skipped": [] }
```

Mode `--json`, `--schema=sales` (agregat, `scope` bernilai `schema`):

```json
{ "command": "push", "scope": "schema", "table_count": 1, "total_rows": 12,
  "tables": [
    { "schema": "sales", "table": "order_item", "rows": 12, "committed_batches": 1 }
  ],
  "skipped": [] }
```

`table_count` menghitung tabel yang benar-benar di-push; tabel yang dilewati tercatat
terpisah di `skipped[]`.

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Push sukses |
| `1` | File input tidak ada (`--table`), envelope invalid / gagal parse, atau column mismatch vs SDF |
| `2` | `--config` wajib namun tidak ada default; bukan tepat satu dari `--table` / `--schema` / `--all-schemas`; `--schema` 0-match (tidak ada tabel cocok); atau tabel tidak terdaftar / ambigu di SDF |
| `3` | Database error (koneksi atau INSERT, termasuk konflik primary key saat append, dan FK violation pada child bila parent di-skip) |

## Contoh

```bat
npx restforge data push --table=visitors
npx restforge data push --schema=public --config=db.env
npx restforge data push --all-schemas --config=db.env --json
npx restforge data push --table=visitors --config=db.env --batch-size=500
npx restforge data push --table=sales.order_item --config=db.env
npx restforge data push --table=visitors --config=db.env --json
```

| Skenario | Command |
|----------|---------|
| Push satu tabel dari `data-storage/` | `data push --table=visitors` |
| Push semua tabel pada satu schema (urut FK) | `data push --schema=sales` |
| Push seluruh tabel SDF lintas schema (urut FK parent→child) | `data push --all-schemas` |
| Push ke database tujuan berbeda (cross-dialect) | `data push --table=visitors --config=db-mysql.env` |
| Atur ukuran batch / commit | `data push --table=visitors --batch-size=500` |
| Tabel ber-schema (file bersarang `sales/order_item.json`) | `data push --table=sales.order_item` |

---

**Lihat juga**: [`README`](./README.md) · [`pull.md`](./pull.md) · [`conventions`](../conventions.md) · [`catalogs/sdf/`](../../../catalogs/sdf/)
