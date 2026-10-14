# `processor list`

> Menampilkan daftar processor yang sudah di-generate di project, beserta file payload dan function processor di dalamnya.

## Pattern

```
npx restforge processor list [options]
```

Daftar diambil dari folder `metadata/` di working directory, yang diisi oleh `processor create`.

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--format <FORMAT>` | Tidak | `text` | Format output: `text` (tabel) atau `json` |
| `--name <PATTERN>` | Tidak | - | Filter nama processor, case-insensitive. Tanda `*` cocok dengan karakter apa pun, sedangkan nama tanpa `*` harus sama persis |

## Output

Output `text` berupa tabel dengan tiga kolom. Kolom Payload berisi nama file `payload/<processor>.json` bila file itu ada, dan `-` bila tidak ada. Kolom Processor berisi function processor beserta HTTP method-nya.

```
Registered Processors:

Processor Name          Payload                     Processor
------------------------------------------------------------------------------------------------
order-validator         order-validator.json        validate-order (POST), order-summary (GET)

Total: 1 processor(s)
```

Processor yang dibuat dengan versi RESTForge lama tampil dengan kolom Processor berisi `-`. Jalankan ulang `processor create` untuk processor tersebut agar daftar function-nya tercatat.

Output `json` memuat field `summary` (jumlah processor dan daftar project) dan array `processors`. Setiap elemen array berisi `project`, `name`, `payload`, dan `processor`.

## Contoh

```bat
npx restforge processor list
npx restforge processor list --format=json
npx restforge processor list --name=*order*
```

---

**Lihat juga**: [`processor/`](./) · [`commands/`](../) · [`README`](../../../README.md)
