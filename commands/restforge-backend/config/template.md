# `config template`

> Output raw template dari `db-connection.env` (format KEY=VALUE).

## Pattern

```
npx restforge config template
```

Command ini tidak menerima flag tambahan.

## Contoh

```bat
npx restforge config template > config/db-connection.env
```

## Konfigurasi Zona dan Format Tanggal-Waktu

Template memuat tiga kunci berkomentar untuk mengatur perilaku field `date`,
`timestamp`, dan `timestamptz`:

| Kunci | Default | Keterangan |
|-------|---------|-----------|
| `TIMEZONE` | `UTC` | Zona IANA (mis. `Asia/Jakarta`) untuk menafsirkan input tanpa zona dan mengisi kolom audit |
| `DATEFORMAT` | `yyyy-MM-dd` | Pola input/output semua field `date` |
| `DATETIMEFORMAT` | `yyyy-MM-dd HH:mm:ss.SSS` | Pola input/output semua field `timestamp`, berbentuk `<pola tanggal> <pola waktu>`. Untuk `timestamptz` hanya berlaku sebagai input lokal; output-nya selalu ISO UTC |

Ketiga kunci ini berlaku sama untuk semua endpoint dan semua field; tidak ada pola per field.
Setiap respons API membawa nilainya di header `X-Timezone`, `X-Date-Format`, dan
`X-DateTime-Format`.

Frontend hasil generate memakai pola yang sama lewat `appConfig.dateFormat` dan
`appConfig.dateTimeFormat`, yang disalin saat `payload migrate`. Setelah `DATEFORMAT` atau
`DATETIMEFORMAT` diubah, jalankan `payload migrate` dan generate frontend ulang.

Nilai yang tidak valid membuat server gagal start dengan pesan yang jelas. Detail
kontrak lengkap ada di [`catalogs/rdf/datetime-fields.md`](../../../catalogs/rdf/datetime-fields.md#konfigurasi-zona-dan-format).

## Batas Jumlah Baris per Permintaan

Bagian Query Limits pada template memuat dua kunci aktif dengan nilai default:

| Kunci | Default | Keterangan |
|-------|---------|-----------|
| `READ_MAX_LIMIT` | `5000` | Batas baris endpoint `/read` mode non-paginasi. Nilai `unlimited` mengizinkan client meminta seluruh baris dengan `limit: "all"` |
| `DATATABLES_MAX_LENGTH` | `1000` | Batas parameter `length` endpoint `/datatables` |

Permintaan di atas batas tidak ditolak. Jumlah baris dipotong ke batas dan response menandainya dengan `truncated: true`. Nilai yang tidak valid (bukan angka, `0`, atau negatif) jatuh ke default. Perilaku lengkap, termasuk trade-off mode `unlimited`, ada di [`endpoint-read.md`](../../../api-spec/endpoint-read.md#batas-jumlah-baris-mode-non-paginasi) dan [`endpoint-datatables.md`](../../../api-spec/endpoint-datatables.md#batas-jumlah-baris-per-permintaan).

---

**Lihat juga**: [`config/`](./) · [`commands/`](../) · [`README`](../../../README.md)
