# `endpoint list`

> Menampilkan daftar endpoint REST dan dashboard yang sudah di-generate di project, beserta file payload dan action yang aktif.

## Pattern

```
npx restforge endpoint list [options]
```

Daftar diambil dari folder `metadata/` di working directory, yang diisi oleh `endpoint create` dan `dashboard create`. Processor tidak ikut tampil di daftar ini dan bisa dilihat lewat [`processor list`](../processor/list.md).

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--format <FORMAT>` | Tidak | `text` | Format output: `text` (tabel) atau `json` |
| `--name <PATTERN>` | Tidak | - | Filter nama endpoint, case-insensitive. Tanda `*` cocok dengan karakter apa pun, sedangkan nama tanpa `*` harus sama persis |

## Output

Output `text` berupa tabel dengan tiga kolom. Kolom Payload berisi nama file `payload/<endpoint>.json` bila file itu ada, dan `-` bila tidak ada. Kolom Action berisi action yang aktif, sedangkan dashboard ditampilkan dengan `-`.

```
Registered Endpoints:

Endpoint Name           Payload                     Action
------------------------------------------------------------------------------------------------
products                products.json               create, read, update, delete,
                                                    datatables

Total: 1 endpoint(s)
```

Output `json` memuat field `summary` (jumlah endpoint dan daftar project) dan array `endpoints`. Setiap elemen array berisi `project`, `name`, `payload`, dan `actions`.

## Contoh

```bat
npx restforge endpoint list
npx restforge endpoint list --format=json
npx restforge endpoint list --name=*product*
```

---

**Lihat juga**: [`endpoint/`](./) · [`commands/`](../) · [`README`](../../../README.md)
