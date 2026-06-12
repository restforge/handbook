# `restforge-designer validate`

> Validasi UDF (file payload JSON) terhadap schema plugin tanpa write file output.

## Sintaks

```bash
restforge-designer validate --payload <PAYLOAD> [OPTIONS]
```

## Flag

| Flag | Tipe | Wajib | Keterangan |
|------|------|:-----:|-----------|
| `-p`, `--payload <PAYLOAD>` | path | ✓ | Path ke file UDF JSON yang akan divalidasi |
| `--plugins-dir <PATH>` | string | ✗ | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |
| `--license <KEY>` | string | ✗ | License key override per-invocation |

## Contoh Penggunaan

### Validasi Standar

```bash
restforge-designer validate --payload=payload/contact-mgmt.json
```

Output sukses:

```
╭─ Validation Successful ─────────────────╮
│ Payload : payload/contact-mgmt.json     │
│ Plugin  : vanilla-js-basic              │
│ Pages   : 3                             │
╰─────────────────────────────────────────╯
```

Output dengan warning:

```
╭─ Validation Successful with Warnings ───────────────╮
│ Payload : payload/contact-mgmt.json                 │
│ Plugin  : vanilla-js-basic                          │
│ Pages   : 3                                         │
│                                                     │
│ Warnings:                                           │
│   - [contact] displayField 'contact_name' is not    │
│     found in the 'fields' definition                │
│   - [contact] Field 'email' has inTable:true but    │
│     no tableOrder is defined                        │
╰─────────────────────────────────────────────────────╯
```

Output dengan error:

```
╭─ Validation Failed ─────────────────────────────────╮
│ Payload : payload/contact-mgmt.json                 │
│                                                     │
│ Errors:                                             │
│   - appConfig.appName must be provided              │
│   - [contact] pageType 'invalid' is invalid.        │
│     Valid types: crud, dashboard                    │
│   - Field 'is_active' has an invalid type: 'flag'.  │
│     Valid types are: checkbox, date, number,        │
│     select, text, textarea, time, timestamp         │
╰─────────────────────────────────────────────────────╯
```

### Validasi dengan Plugins Dir Custom

```bash
restforge-designer validate --payload=payload/contact-mgmt.json --plugins-dir=./my-plugins
```

### Validasi dengan Override License Per-Invocation

```bash
restforge-designer validate --payload=payload/contact-mgmt.json --license=ABCD-1234-EFGH-5678
```

## Behavior

Verb `validate`:

1. **Membaca** file UDF di path `--payload`
2. **Muat plugin** sesuai field `appConfig.plugin` di UDF
3. **Validasi** struktur UDF terhadap aturan validator universal + aturan spesifik plugin
4. **Tampilkan ringkasan** errors dan warnings ke console
5. **Tidak menulis file apa pun** ke disk

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Validasi sukses (warning tidak menyebabkan failure) |
| `1` | Validasi gagal (ada error), file tidak ditemukan, atau JSON malformed |

## Aturan Validasi yang Dievaluasi

Detail lengkap aturan validasi UDF (mandatory fields, pattern validation, cross-field checks) ada di [`catalogs/udf/validation-rules.md`](../../catalogs/udf/validation-rules.md).

Ringkasan kategori validasi:

| Kategori | Contoh Aturan |
|----------|---------------|
| Envelope | `appConfig`, `pages` wajib |
| App config | `appName`, `appCode`, `plugin`, `apiBaseUrl` wajib |
| Page CRUD | `pageId` pattern, `pageType` valid, `fields` minimal 1 |
| Page dashboard | `dataSources`, `rows` wajib, field CRUD-only dilarang |
| Field | `name`, `label`, `type` wajib, `type` di whitelist |
| Navigation | `pageRef` ada di `pages`, depth maks 3 |
| Live sync | URL scheme `ws://`/`wss://`, konsistensi dengan `features.enableLiveSync` |

## Perbedaan dengan `preview`

| Aspek | `validate` | `preview` |
|-------|-----------|-----------|
| Run validator | ✓ | ✓ (sebelum preview) |
| Output ke console | Error + warning saja | Error + warning + daftar file yang akan di-generate |
| License check | Tidak diperlukan | Diperlukan |
| Use case | Quick check setelah edit UDF | Verifikasi file output sebelum generate |

## Use Case

| Skenario | Rekomendasi |
|----------|-------------|
| Edit UDF, mau cepat cek struktur | `validate` |
| Mau lihat file apa saja yang akan di-generate | `preview` |
| Mau langsung generate aplikasi | `generate` (sudah include validate internally) |
| CI/CD pre-merge check | `validate` (cepat, tidak butuh license) |

---

**Lihat juga**: [`README`](./README.md) · [`preview.md`](./preview.md) · [`generate.md`](./generate.md) · [`catalogs/udf/validation-rules.md`](../../catalogs/udf/validation-rules.md)
