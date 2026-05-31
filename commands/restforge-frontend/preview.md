# `restforge-designer preview`

> Preview daftar file hasil generate tanpa write ke disk.

## Sintaks

```bash
restforge-designer preview --payload <PAYLOAD> [OPTIONS]
```

## Flag

| Flag | Tipe | Wajib | Keterangan |
|------|------|:-----:|-----------|
| `-p`, `--payload <PAYLOAD>` | path | ✓ | Path ke file UDF JSON |
| `--plugins-dir <PATH>` | string | ✗ | Override folder plugins (env: `RESTFORGE_DESIGNER_PLUGINS_DIR`) |
| `--license <KEY>` | string | ✗ | License key override per-invocation |

## Contoh Penggunaan

### Preview Standar

```bash
restforge-designer preview --payload=payload/contact-mgmt.json
```

Output yang umum:

```
╭─ Generation Preview ────────────────────────────────╮
│ Payload : payload/contact-mgmt.json                 │
│ Plugin  : vanilla-js-basic                          │
│ Pages   : 3                                         │
│                                                     │
│ Files that would be generated (24):                 │
│                                                     │
│ Shared files:                                       │
│   index.html                                        │
│   app-start.bat                                     │
│   js/common.js                                      │
│   css/global-list.css                               │
│                                                     │
│ Per-page files:                                     │
│   contact.html              + js/contact.js         │
│   supplier.html             + js/supplier.js        │
│   product.html              + js/product.js         │
│                                                     │
│ Plugin assets:                                      │
│   vendor/jquery.min.js                              │
│   vendor/select2/select2.min.js                     │
│   ...                                               │
╰─────────────────────────────────────────────────────╯
```

### Preview dengan Override License

```bash
restforge-designer preview --payload=payload/contact-mgmt.json --license=ABCD-1234-EFGH-5678
```

### Preview dengan Plugins Dir Custom

```bash
restforge-designer preview --payload=payload/contact-mgmt.json --plugins-dir=./my-plugins
```

## Behavior

Verb `preview`:

1. **Validasi license** terlebih dahulu (verb ini memerlukan license aktif)
2. **Membaca** file UDF di path `--payload`
3. **Load plugin** sesuai field `appConfig.plugin`
4. **Jalankan validator** UDF
5. **Hitung daftar file** yang akan dihasilkan generator (tanpa write ke disk)
6. **Tampilkan** daftar file ke console

## Exit Code

| Exit Code | Kondisi |
|-----------|---------|
| `0` | Preview sukses |
| `1` | Validasi gagal, license invalid, file tidak ditemukan, atau plugin tidak ditemukan |

## Kategori File yang Di-list

Daftar file dikelompokkan per kategori:

| Kategori | Isi |
|----------|-----|
| Shared files | File aplikasi-level: `index.html`, `app-start.bat`, `js/common.js`, `css/global-list.css` |
| Per-page files | File per page: `{pageId}.html` + `js/{pageId}.js` |
| Plugin assets | File yang di-copy dari plugin (vendor JS, CSS, dll.) |

Jumlah dan jenis file tergantung pada plugin yang dipakai. Plugin `vanilla-js-auth` menambahkan file auth (login.html, JWT helper) yang tidak ada di `vanilla-js-basic`.

## Use Case

| Skenario | Rekomendasi |
|----------|-------------|
| Mau lihat file output sebelum overwrite folder existing | `preview` |
| Quick check jumlah file yang akan di-generate | `preview` |
| Debug "kenapa file X tidak ada" | `preview` + bandingkan dengan output `generate` |
| CI dry-run pre-merge | `preview` (memerlukan license — jika tidak ada license di CI, pakai `validate`) |

## Perbedaan dengan `validate` dan `generate`

| Aspek | `validate` | `preview` | `generate` |
|-------|-----------|-----------|-----------|
| Run validator UDF | ✓ | ✓ | ✓ |
| Output ke console | Error + warning | Error + warning + daftar file | Error + warning + ringkasan files written |
| License diperlukan | Tidak | Ya | Ya |
| Write file ke disk | Tidak | Tidak | Ya |

---

**Lihat juga**: [`README`](./README.md) · [`validate.md`](./validate.md) · [`generate.md`](./generate.md)
