# Upload — Endpoint File ke Object Storage

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| **Upload langsung** | `POST /upload?field={field}` — kirim file, terima metadata |
| **Hapus file** | `POST /upload-delete` — hapus satu file dari storage |
| **Presigned URL** | `GET /upload-url` — minta URL upload langsung ke storage |
| **Konfirmasi presigned** | `POST /upload-confirm` — verifikasi file lalu susun metadata |
| **Cleanup orphan** | `POST /upload-cleanup` — hapus file yang tidak pernah dipakai record |
| **Format Input** | `multipart/form-data` untuk `/upload`, JSON untuk endpoint lain |
| **Storage** | AWS S3, S3-compatible, atau local filesystem |
| **Database** | Tidak ada operasi database pada seluruh endpoint di halaman ini |
| **Aktivasi** | `action.upload: true` dan blok `uploadConfig` pada RDF |

---

## Ikhtisar

Endpoint upload menangani file binary saja. Metadata hasil upload dikembalikan ke client, lalu client menyimpannya ke record lewat `/create` atau `/update`. Pemisahan ini serta alur panggilan lengkapnya dijelaskan di [`features/file-storage/`](../features/file-storage/README.md).

Endpoint `/upload-url` dan `/upload-confirm` hanya terdaftar bila `uploadConfig.presignedUrl` bernilai `true`. Tiga endpoint lainnya selalu terdaftar begitu resource memiliki field upload.

---

## Upload Langsung (POST /upload)

### Format Request

**Content-Type:** `multipart/form-data`

```
POST /api/{project}/{endpoint}/upload?field=photos
```

| Parameter | Lokasi | Wajib | Keterangan |
|-----------|--------|:-----:|------------|
| `field` | Query string atau body | **Ya** | Nama field tujuan, harus terdaftar di `uploadConfig.fields` |
| `files` | Form field | **Ya** | Satu atau beberapa file. Jumlah maksimum mengikuti `maxFiles` |
| `x-user-id` | Header | Tidak | Identitas user yang dicatat pada metadata sebagai `uploadedBy` |

### Response Sukses

**HTTP Status:** `201 Created`

```json
{
  "success": true,
  "message": "2 file(s) uploaded successfully",
  "data": {
    "fieldName": "photos",
    "uploaded": [
      {
        "key": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg",
        "originalName": "foto produk.jpg",
        "fileName": "0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg",
        "contentType": "image/jpeg",
        "size": 245760,
        "url": "https://my-bucket.s3.ap-southeast-1.amazonaws.com/mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg",
        "provider": "s3",
        "bucket": "my-bucket",
        "uploadedAt": "2026-08-21T04:12:33.128Z"
      }
    ]
  },
  "timestamp": "2026-08-21T04:12:33.140Z"
}
```

Isi array `uploaded` dikirim apa adanya ke `/create` atau `/update` sebagai nilai field terkait.

### Response Error

| HTTP Status | Kondisi |
|-------------|---------|
| `400` | Parameter `field` tidak dikirim |
| `400` | Field tidak terdaftar di `uploadConfig.fields` |
| `400` | Tidak ada file pada request |
| `400` | Extension atau MIME type di luar daftar yang diizinkan |
| `400` | Ukuran file melampaui `maxFileSize`, atau jumlah file melampaui `maxFiles` |
| `500` | Kegagalan saat mengirim file ke storage |

Bila salah satu file gagal validasi, seluruh file pada request tersebut dibatalkan dan file sementara dihapus.

---

## Hapus File (POST /upload-delete)

Menghapus satu file dari storage. Metadata pada record tidak ikut berubah, sehingga client tetap perlu memanggil `/update` dengan array yang sudah dikurangi.

### Format Request

```json
{
  "fieldName": "photos",
  "fileKey": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg"
}
```

| Field | Tipe | Wajib | Keterangan |
|-------|------|:-----:|------------|
| `fieldName` | string | **Ya** | Nama field tujuan, harus terdaftar di `uploadConfig.fields` |
| `fileKey` | string | **Ya** | Nilai `key` dari metadata file |

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "File deleted from storage successfully",
  "data": {
    "fieldName": "photos",
    "deletedKey": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg"
  },
  "timestamp": "2026-08-21T04:18:02.551Z"
}
```

### Response Error

| HTTP Status | Kondisi |
|-------------|---------|
| `400` | `fieldName` atau `fileKey` tidak dikirim |
| `400` | Field tidak terdaftar di `uploadConfig.fields` |
| `500` | Kegagalan penghapusan di storage |

---

## Presigned URL (GET /upload-url)

Menghasilkan URL bertanda tangan agar client mengirim file langsung ke storage tanpa melewati server aplikasi. Endpoint ini hanya tersedia bila `uploadConfig.presignedUrl` bernilai `true`.

### Format Request

```
GET /api/{project}/{endpoint}/upload-url?field=photos&filename=foto.jpg&contentType=image/jpeg
```

| Parameter | Wajib | Keterangan |
|-----------|:-----:|------------|
| `field` | **Ya** | Nama field tujuan |
| `filename` | **Ya** | Nama file asli, dipakai untuk membentuk storage key dan memeriksa extension |
| `contentType` | Tidak | MIME type file. Default `application/octet-stream` |

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "data": {
    "uploadUrl": "https://my-bucket.s3.ap-southeast-1.amazonaws.com/mini-inventory/...?X-Amz-Signature=...",
    "key": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto.jpg",
    "expiresIn": 3600
  },
  "timestamp": "2026-08-21T04:20:11.004Z"
}
```

Client mengirim file dengan metode `PUT` ke `uploadUrl`, lalu memanggil `/upload-confirm` memakai nilai `key`. Masa berlaku URL diatur lewat `UPLOAD_PRESIGNED_EXPIRY`.

### Response Error

| HTTP Status | Kondisi |
|-------------|---------|
| `400` | Parameter `field` atau `filename` tidak dikirim |
| `400` | Field tidak terdaftar, atau extension di luar `allowedTypes` |
| `501` | Provider storage aktif tidak mendukung presigned URL, misalnya provider `local` |
| `500` | Kegagalan pembuatan signature |

---

## Konfirmasi Presigned (POST /upload-confirm)

Memverifikasi bahwa file hasil upload presigned benar-benar ada di storage, lalu menyusun metadata dengan struktur yang sama seperti `/upload`.

### Format Request

```json
{
  "fieldName": "photos",
  "key": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto.jpg"
}
```

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "Upload confirmed successfully",
  "data": {
    "fieldName": "photos",
    "confirmed": {
      "key": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto.jpg",
      "originalName": "0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto.jpg",
      "fileName": "0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto.jpg",
      "contentType": "image/jpeg",
      "size": 245760,
      "url": "https://my-bucket.s3.ap-southeast-1.amazonaws.com/mini-inventory/...",
      "provider": "s3",
      "bucket": "my-bucket",
      "uploadedAt": "2026-08-21T04:21:47.900Z"
    }
  },
  "timestamp": "2026-08-21T04:21:47.912Z"
}
```

Nilai `originalName` diturunkan dari segmen terakhir key, karena nama file asli tidak ikut terkirim pada upload presigned.

### Response Error

| HTTP Status | Kondisi |
|-------------|---------|
| `400` | `fieldName` atau `key` tidak dikirim |
| `400` | Field tidak terdaftar di `uploadConfig.fields` |
| `404` | File tidak ditemukan di storage, misalnya upload gagal atau URL sudah kedaluwarsa |
| `500` | Kegagalan pembacaan metadata dari storage |

---

## Cleanup Orphan (POST /upload-cleanup)

Memicu pembersihan file orphan secara manual, memakai logika yang sama dengan job terjadwal. File orphan adalah file yang sudah masuk storage namun metadatanya tidak pernah tersimpan ke record.

### Format Request

Request tidak memerlukan body.

```
POST /api/{project}/{endpoint}/upload-cleanup
```

### Response Sukses

**HTTP Status:** `200 OK`

```json
{
  "success": true,
  "message": "Cleanup completed: 3 file(s) deleted, 0 failed",
  "data": {
    "deleted": 3,
    "failed": 0
  },
  "timestamp": "2026-08-21T04:25:09.377Z"
}
```

Bila provider storage tidak tersedia, response tetap HTTP 200 dengan `success: false` dan pesan bahwa cleanup dilewati.

### Catatan Cakupan

| Aspek | Perilaku |
|-------|----------|
| Cakupan | Seluruh file pending pada project, bukan hanya resource tempat endpoint dipanggil |
| Ambang umur | File pending yang lebih muda dari `UPLOAD_CLEANUP_THRESHOLD` tidak disentuh |
| Dependency | Membutuhkan Redis, karena daftar file pending disimpan di sana |
| Kegagalan penghapusan | Key tetap berada di daftar pending dan dicoba lagi pada siklus berikutnya |

---

## Referensi Silang

| Dokumen | Lokasi | Keterangan |
|---------|--------|------------|
| File Storage | [../features/file-storage/](../features/file-storage/README.md) | Konfigurasi provider, alur kerja, struktur metadata |
| `uploadConfig` | [../catalogs/rdf/upload-config.md](../catalogs/rdf/upload-config.md) | Spec property RDF untuk mengaktifkan upload |
| Auth Guard | [../catalogs/rdf/auth-guard.md](../catalogs/rdf/auth-guard.md) | Pemetaan permission endpoint upload |
