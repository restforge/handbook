# `data pull`

> Export isi tabel database ke file envelope JSON (`data-storage/<table>.json`)
> berdasarkan metadata SDF. Mendukung satu tabel (`--table`), per schema (`--schema`),
> atau seluruh tabel terdaftar (`--all-schemas`).

Kolom yang diekspor dan tipenya ditentukan **murni dari SDF** (bukan introspeksi
database). Hanya tabel yang terdaftar di SDF yang dapat di-pull. Penjelasan format
file, resolusi config, penamaan file, dan penanganan lintas-dialect ada di
[`README.md`](./README.md).

## Pattern

```
npx restforge data pull --table=<NAME> [options]
npx restforge data pull --schema=<LIST> [options]
npx restforge data pull --all-schemas [options]
```

## Flag

| Flag | Tipe | Default | Wajib | Keterangan |
|------|------|---------|:-----:|-----------|
| `--table=<NAME>` | string | — | ✗* | Nama tabel sumber (harus terdaftar di SDF). Bentuk `schema.table` untuk tabel ber-schema. Mutual-exclusive dengan `--schema` / `--all-schemas` |
| `--schema=<LIST>` | string | — | ✗* | Filter schema: satu nama atau comma-separated (mis. `public,sales`). Mem-pull semua tabel SDF pada schema tersebut. Mutual-exclusive dengan `--table` / `--all-schemas` |
| `--all-schemas` | flag | `false` | ✗* | Pull seluruh tabel yang terdaftar di `--schema-path`, lintas semua schema. Mutual-exclusive dengan `--table` / `--schema` |
| `--config=<FILE>` | path | default `.restforge/defaults.json` | ✗ | File config database (`.env`). Wajib bila tidak ada default |
| `--schema-path=<PATH>` | path | `schema` | ✗ | Lokasi SDF (file atau folder) sumber metadata tabel |
| `--format=<FMT>` | string | `json` | ✗ | Format penyimpanan nilai; saat ini hanya menerima `json` |
| `--limit=<N>` | number | — (seluruh tabel) | ✗ | Batas maksimum total rows yang di-export (diterapkan per tabel pada `--schema` / `--all-schemas`) |
| `--batch-size=<N>` | number | `1000` | ✗ | Ukuran batch baca internal dari database |
| `--storage-path=<DIR>` | path | `data-storage` | ✗ | Folder output relatif terhadap working directory |
| `--force` | flag | `false` | ✗ | Overwrite file output bila sudah ada (berlaku ke semua file pada `--schema` / `--all-schemas`) |
| `--json` | flag | `false` | ✗ | Output ringkasan dalam format JSON (untuk konsumsi CI/CD) |

\* Tepat satu dari `--table` / `--schema` / `--all-schemas` wajib disertakan.

> **Mutual-exclusive**: tepat satu dari `--table`, `--schema`, atau `--all-schemas` wajib
> disertakan. Menyebut lebih dari satu, atau tidak menyebut satupun, ditolak sebagai usage
> error (exit `2`) sebelum koneksi database dibuka.

## Behavior

1. **Resolve config** database (lihat [README — Resolusi Config](./README.md#resolusi-config-config-resolution)).
2. **Validasi mutual-exclusive**: tepat satu dari `--table` / `--schema` / `--all-schemas`;
   bila lebih dari satu atau tidak ada satupun disebut → exit `2`, sebelum koneksi database.
3. **Validasi `--format`** (hanya `json` yang didukung; nilai lain → exit `2`).
4. **Load SDF + table-guard** sebelum koneksi database. Metadata kolom, namespace, dan
   primary key seluruhnya dari SDF IR. Pada `--table`, tabel tidak terdaftar / ambigu →
   ditolak (exit `2`). Pada `--schema`, bila tidak ada satupun tabel SDF yang cocok dengan
   schema yang diminta (0-match) → ditolak (exit `2`). Pada `--all-schemas`, SDF tanpa tabel
   sama sekali menghasilkan 0 tabel ter-proses dan exit `0` (benign).

Langkah berikut dijalankan **per tabel** (sekali pada `--table`, untuk tiap tabel hasil scope pada `--schema` / `--all-schemas`):

5. **Cek file output**: bila sudah ada dan tanpa `--force`, dibatalkan sebelum konek
   database (exit `1`).
6. **Connect** + validasi identifier tabel/kolom (anti-injection).
7. **SELECT bertahap**: keyset pagination bila primary key tunggal (`ORDER BY pk`,
   `WHERE pk > last`), fallback OFFSET bila primary key composite. `--limit` membatasi
   total rows per tabel.
8. **Tulis envelope streaming**: header lalu rows per batch ditulis langsung ke file
   (tidak menahan seluruh data di memory), cocok untuk tabel besar. Nilai di-encode per
   kolom mengikuti tabel [Representasi Nilai](./README.md#representasi-nilai-value-representation).
9. **Disconnect** + tampilkan ringkasan.

Bila terjadi error di tengah penulisan, file partial dihapus agar tidak meninggalkan
envelope tidak lengkap.

### Mode Multi-Tabel (`--schema` / `--all-schemas`)

`--schema=<LIST>` mem-pull setiap tabel SDF yang termasuk pada schema yang diminta,
sedangkan `--all-schemas` mem-pull **setiap tabel** yang terdaftar di `--schema-path`
lintas semua schema. Keduanya menulis satu file per tabel ke folder `--storage-path`.
Karakteristiknya:

- **Satu file per tabel, layout bersarang per schema**: tabel ber-schema ditulis ke
  subfolder bernama schema-nya (`<schema>/<table>.json`), tabel tanpa schema ditulis flat
  di root (lihat [README — Nama File Output](./README.md#nama-file-output-output-file-naming)).
- **`--limit` per tabel**: batas diterapkan independen pada tiap tabel, bukan total agregat.
- **`--force` ke semua**: overwrite berlaku pada seluruh file output.
- **Fail-fast**: bila satu tabel gagal, proses berhenti pada tabel tersebut. Tabel yang
  sudah selesai sebelum error **tetap tertulis** di disk karena tidak ada rollback
  lintas-file.
- **Urutan**: mengikuti urutan tabel di SDF (file SDF ter-sort alfabetis).
- **Validasi scope**: `--schema` yang tidak cocok dengan tabel manapun (0-match) ditolak
  sebagai usage error (exit `2`); `--all-schemas` pada SDF kosong menghasilkan 0 tabel dan
  exit `0` (benign).

Mode human mencetak satu baris per tabel diakhiri ringkasan agregat, sedangkan mode
`--json` mengeluarkan satu object agregat.

### Output

Mode human (default), `--table`:

```
Pulled 1280 rows from visitors -> data-storage/visitors.json (dialect: postgresql)
```

Mode `--json`, `--table`:

```json
{ "command": "pull", "schema": null, "table": "visitors", "rows": 1280, "file": "data-storage/visitors.json", "source_dialect": "postgresql" }
```

Mode human, `--all-schemas`:

```
Pulled 4 rows from visitor_categories -> data-storage/visitor_categories.json (dialect: postgresql)
Pulled 4 rows from visitors -> data-storage/visitors.json (dialect: postgresql)
Pulled 12 rows from sales.order_item -> data-storage/sales/order_item.json (dialect: postgresql)
Done: 3 tables, 20 rows total
```

Mode `--json`, `--all-schemas` (agregat):

```json
{ "command": "pull", "scope": "all-schemas", "table_count": 3, "total_rows": 20,
  "tables": [
    { "schema": null,    "table": "visitor_categories", "rows": 4,  "file": "data-storage/visitor_categories.json", "source_dialect": "postgresql" },
    { "schema": null,    "table": "visitors",            "rows": 4,  "file": "data-storage/visitors.json",            "source_dialect": "postgresql" },
    { "schema": "sales", "table": "order_item",          "rows": 12, "file": "data-storage/sales/order_item.json",     "source_dialect": "postgresql" }
  ] }
```

Mode `--json`, `--schema=sales` (agregat, `scope` bernilai `schema`):

```json
{ "command": "pull", "scope": "schema", "table_count": 1, "total_rows": 12,
  "tables": [
    { "schema": "sales", "table": "order_item", "rows": 12, "file": "data-storage/sales/order_item.json", "source_dialect": "postgresql" }
  ] }
```

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Pull sukses |
| `1` | File output sudah ada tanpa `--force`, atau gagal baca SDF (path tidak ada / parse gagal) |
| `2` | `--config` wajib namun tidak ada default; bukan tepat satu dari `--table` / `--schema` / `--all-schemas`; `--schema` 0-match (tidak ada tabel cocok); tabel tidak terdaftar / ambigu di SDF; atau `--format` selain `json` |
| `3` | Database error (koneksi atau query SELECT) |

## Contoh (Examples)

```bat
npx restforge data pull --table=visitors
npx restforge data pull --schema=public --config=db.env
npx restforge data pull --all-schemas --config=db.env --json
npx restforge data pull --table=visitors --config=db.env --limit=500
npx restforge data pull --table=sales.order_item --config=db.env
npx restforge data pull --table=visitors --config=db.env --force
npx restforge data pull --table=visitors --config=db.env --json
```

| Skenario | Command |
|----------|---------|
| Pull satu tabel ke `data-storage/` | `data pull --table=visitors` |
| Pull semua tabel pada satu schema | `data pull --schema=sales` |
| Pull seluruh tabel SDF lintas schema | `data pull --all-schemas` |
| Pull terbatas 500 baris (per tabel) | `data pull --table=visitors --limit=500` |
| Timpa file yang sudah ada | `data pull --table=visitors --force` |
| Tabel ber-schema (file bersarang `sales/order_item.json`) | `data pull --table=sales.order_item` |

---

**Lihat juga**: [`README`](./README.md) · [`push.md`](./push.md) · [`conventions`](../conventions.md) · [`catalogs/sdf/`](../../../catalogs/sdf/)
