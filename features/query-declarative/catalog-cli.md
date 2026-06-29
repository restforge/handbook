# Introspeksi via CLI (CLI Introspection)

> Command `catalog query-declarative` untuk discovery spec query secara programmatic.

Kembali ke [ikhtisar Query Declarative](README.md).

Command `catalog query-declarative` berfungsi untuk mengeluarkan JSON catalog yang berisi query properties, endpoint resolution, dan file reference convention. Output ini menjadi rujukan tunggal bagi AI agent, generator dokumentasi, dan validator payload. Reference command ada di [`commands/catalog/query-declarative.md`](../../commands/restforge-backend/catalog/query-declarative.md).

---

## Menampilkan Catalog (Showing the Catalog)

Jalankan command tanpa flag untuk mendapatkan full catalog dalam format pretty-printed.

```bash
npx restforge catalog query-declarative
```

Command hanya menulis JSON ke stdout tanpa side effect, sehingga aman dipanggil oleh subprocess atau di-pipe ke tool lain. Output selalu berupa full catalog dengan struktur berikut.

```json
{
  "schemaVersion": "1.0",
  "source": "query-declarative-catalog",
  "summary": {
    "totalQueryProperties": 5,
    "totalEndpoints": 7,
    "totalDatabasePlaceholders": 3,
    "totalFileReferenceCapableProperties": 4
  },
  "queryProperties": [ ... ],
  "endpointResolution": [ ... ],
  "fileReferenceConvention": { ... },
  "documentationUrl": "https://restforge.dev/docs/server/query-filtering/query-declarative"
}
```

---

## Parameter (Parameters)

Command mengikuti pattern standar `npx restforge <resource> <verb> [options]`, dengan `catalog` sebagai resource dan `query-declarative` sebagai verb.

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|------------|
| `--pretty=<true\|false>` | Tidak | `true` | Pretty-print output JSON. Tetapkan `false` untuk compact single-line JSON |
| `--help`, `-h` | Tidak | - | Tampilkan help text |

> **Catatan:** Versi sebelumnya (`restforge-cli query-declarative:catalog`) menyediakan flag filter `--section`, `--name`, `--endpoint`, `--supports-file-reference`, dan `--required`. Flag tersebut sudah tidak tersedia. Command sekarang selalu mengeluarkan full catalog, dan filtering dilakukan di sisi client dengan `jq`.

---

## Memfilter Output dengan jq (Filtering with jq)

Karena command selalu mengeluarkan full catalog, gunakan `jq` untuk mengambil bagian atau item spesifik. Pakai `--pretty=false` agar output siap di-pipe.

Daftar nama property:

```bash
npx restforge catalog query-declarative --pretty=false | jq '.queryProperties[] | .name'
```

Detail satu property:

```bash
npx restforge catalog query-declarative --pretty=false | jq '.queryProperties[] | select(.name == "detailQuery")'
```

Property yang melayani endpoint tertentu:

```bash
npx restforge catalog query-declarative --pretty=false | jq '.queryProperties[] | select(.endpointsServed | index("/read"))'
```

Resolution rule untuk satu endpoint:

```bash
npx restforge catalog query-declarative --pretty=false | jq '.endpointResolution[] | select(.endpoint == "/lookup")'
```

File reference convention:

```bash
npx restforge catalog query-declarative --pretty=false | jq '.fileReferenceConvention'
```

---

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| 0 | Sukses |
| ≠ 0 | Error: nilai boolean tidak valid atau opsi tidak dikenal |

---

**Lihat juga**: [`commands/catalog/query-declarative.md`](../../commands/restforge-backend/catalog/query-declarative.md) · [`commands/query/validate.md`](../../commands/restforge-backend/query/validate.md) · [ikhtisar Query Declarative](README.md)
