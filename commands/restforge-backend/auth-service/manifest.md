# `auth-service manifest`

> Menyusun manifest permission dari RDF project aplikasi yang `authGuard`-nya aktif.

Manifest adalah file JSON berisi app code dan daftar resource beserta aksinya. [`provision`](./provision.md) membaca file ini untuk mendaftarkan permission ke layanan auth.

## Pattern

```
npx restforge auth-service manifest --project=<NAME> [OPTIONS]
```

## Flag

| Flag | Wajib | Default | Keterangan |
|------|-------|---------|-----------|
| `--project <NAME>` | Ya | - | Nama project aplikasi |
| `--output <FILE>` | Tidak | `config/auth-manifest.json` | Lokasi file manifest |

## Prasyarat

- [`auth-service link`](./link.md) sudah dijalankan, sehingga RDF project memiliki `authGuard` aktif.

## Apa yang Dikerjakan

1. Memilih RDF project yang `authGuard`-nya aktif.
2. Menurunkan satu resource per endpoint. Nama resource berasal dari nama endpoint dalam huruf besar, dengan `-` menjadi `_`.
3. Menyelaraskan aksi dengan matriks `action` RDF saat ini.
4. Mengisi `description` resource yang belum punya description.
5. Menulis manifest ke `--output`.

Aksi yang diturunkan dari RDF mengikuti tabel berikut:

| Aksi | Action RDF |
|------|-----------|
| `CREATE` | `create`, `createComposite` |
| `READ` | `read`, `readComposite`, `datatables`, `first`, `lookup`, `aggregate` |
| `UPDATE` | `update`, `updateComposite`, `adjust`, `restore`, `upload` |
| `DELETE` | `delete` |
| `EXPORT` | `export` |
| `IMPORT` | `import` |
| `CHANGE_STATUS` | `workflow` |

Bila suatu aksi dimatikan di RDF, aksi itu dilepas dari manifest dan output menampilkan catatan. Permission yang sudah terdaftar lewat `provision` tidak dihapus.

## Description Resource

Resource yang belum punya `description` mendapat nilai awal berbahasa Inggris. Nilainya diambil dari key `description` di RDF resource tersebut. Bila RDF tidak punya `description`, `manifest` menyusun kalimat dari nama resource, misalnya `Manage item records.` untuk resource `ITEM`.

```json
{
  "appCode": "MYAPP",
  "resources": [
    {
      "resource": "ITEM",
      "description": "Manage item records.",
      "actions": ["CREATE", "READ", "UPDATE", "DELETE"]
    }
  ]
}
```

Description resource tampil di baris resource pada halaman Assign Permissions editor [RBAC](../../restforge-frontend/rbac.md#matriks-assign-permissions). Ganti nilainya dengan keterangan yang lebih jelas, lalu jalankan `provision`.

## Menambah Isi Manifest Sendiri

Manifest boleh diedit setelah dibuat. Isi berikut dipertahankan pada `manifest` berikutnya:

- aksi yang tidak diturunkan dari RDF, misalnya `PRINT` atau `APPROVE`
- `categories`, `category`, dan `sortOrder`
- description resource dan description aksi

Description aksi ditulis sebagai object `{ "action": "...", "description": "..." }` di dalam `actions`. `manifest` tidak pernah membuat description aksi.

```json
{
  "appCode": "MYAPP",
  "categories": [{ "name": "Master Data", "sortOrder": 10 }],
  "resources": [
    {
      "resource": "ITEM",
      "description": "Product catalog with price per item.",
      "actions": [
        "CREATE",
        "READ",
        "UPDATE",
        "DELETE",
        { "action": "PRINT", "description": "Print item price labels." }
      ],
      "category": "Master Data"
    }
  ]
}
```

`PRINT` tidak dijaga endpoint mana pun. Permission ini dipakai kode frontend untuk menampilkan atau menyembunyikan tombol cetak.

Resource yang endpoint atau RDF-nya sudah dihapus tetap ada di manifest dan output menampilkan peringatan.

## Contoh

```bat
cd /d D:\projects\myapp
npx restforge auth-service manifest --project=myapp
npx restforge auth-service manifest --project=myapp --output=config/auth-manifest.json
```

## Output

```
Permission manifest (config/auth-manifest.json), appCode MYAPP:
  ITEM (endpoint item): CREATE, READ, UPDATE, DELETE
  added: ITEM

Manifest written: config/auth-manifest.json
Manual actions (e.g. PRINT, APPROVE), categories, category, sortOrder, and description can be added to the file; they are kept on the next run.
Next: npx restforge auth-service provision --manifest=config/auth-manifest.json --owner-email=<owner email>
```

Bila isi manifest tidak berubah, output menampilkan `Manifest unchanged`.

## Error Umum

| Pesan | Penyebab |
|-------|----------|
| `No RDF of project ... has an enabled authGuard` | `link` belum dijalankan untuk project |
| `RDF files of project ... declare different authGuard.appCode values` | RDF memakai app code berbeda. Jalankan `link` lagi agar seragam |
| `Existing manifest ... is not valid JSON` | File manifest rusak. Perbaiki atau hapus file tersebut |

---

**Lihat juga**: [`auth-service/`](./README.md) · [`link`](./link.md) · [`provision`](./provision.md) · [`commands/`](../../README.md) · [`README`](../../../README.md)
