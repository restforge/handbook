# Alur Kerja dan Metadata

> Urutan panggilan untuk menyimpan, mengganti, dan menghapus file, beserta struktur metadata yang tersimpan di database.

Kembali ke [ikhtisar File Storage](README.md).

---

## Prinsip Pemisahan

Endpoint upload tidak pernah menjalankan INSERT, UPDATE, maupun DELETE ke database. Sebaliknya, endpoint CRUD tidak pernah mengirim atau menghapus file di storage, kecuali pada cascade delete yang dijelaskan di bawah.

Pemisahan ini menjaga tiga hal:

- Upload dapat dilakukan sebelum record ada, misalnya pada form create yang belum disimpan.
- Kegagalan penyimpanan record tidak menyisakan file setengah jadi yang tercatat di database.
- Satu file dapat diverifikasi terlebih dahulu, baru metadatanya disusun oleh client sesuai struktur record.

Konsekuensinya, client memegang metadata di antara dua panggilan dan bertanggung jawab mengirimkannya ke endpoint CRUD.

---

## Alur Create Record dengan File

```
1. POST /upload?field=photos          → file masuk ke storage, response berisi metadata
2. POST /create                       → metadata dikirim di body sebagai array pada field photos
```

Contoh body `/create` setelah dua file ter-upload:

```json
{
    "name": "Produk A",
    "photos": [
        { "key": "...", "originalName": "foto-1.jpg", "...": "..." },
        { "key": "...", "originalName": "foto-2.jpg", "...": "..." }
    ]
}
```

Isi array diambil apa adanya dari response `/upload`. Saat `/create` berhasil, key file terkait dikeluarkan dari daftar pending sehingga tidak akan dihapus oleh cleanup orphan.

---

## Alur Update File pada Record Existing

Mengganti file berarti tiga panggilan, karena penghapusan file lama dan penulisan record adalah dua operasi berbeda:

```
1. POST /upload?field=photos          → file baru masuk ke storage
2. POST /upload-delete                → file lama dihapus dari storage
3. POST /update                       → field photos diisi array final hasil susunan client
```

Array yang dikirim ke `/update` adalah array final, bukan selisih perubahan. Client membaca array lama dari record, menghapus entry file yang sudah tidak dipakai, menambahkan entry file baru, lalu mengirim hasilnya.

Urutan langkah 1 dan 2 boleh ditukar. Yang tidak boleh dilewati adalah langkah 3, karena tanpanya database masih menunjuk file yang sudah tidak ada di storage.

---

## Alur Hapus File dari Record

```
1. POST /upload-delete                → file dihapus dari storage
2. POST /update                       → array photos dikirim ulang tanpa entry file tersebut
```

---

## Alur Hapus Record

```
1. POST /delete                       → record dihapus, seluruh file pada field upload ikut dihapus
```

Cascade delete berjalan otomatis untuk setiap field yang terdaftar di `uploadConfig.fields`. Runtime membaca array metadata dari record yang terhapus, mengumpulkan seluruh key, lalu menghapusnya dari storage dalam satu operasi batch.

Cascade bersifat non-blocking: kegagalan penghapusan file dicatat di log dan tidak membatalkan penghapusan record yang sudah berhasil.

---

## Struktur Metadata File

Setiap file diwakili satu object di dalam array. Struktur ini dihasilkan oleh `/upload` dan `/upload-confirm`, dan disimpan apa adanya oleh endpoint CRUD.

```json
{
    "key": "mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg",
    "originalName": "foto produk.jpg",
    "fileName": "0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg",
    "contentType": "image/jpeg",
    "size": 245760,
    "url": "https://my-bucket.s3.ap-southeast-1.amazonaws.com/mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg",
    "provider": "s3",
    "bucket": "my-bucket",
    "uploadedAt": "2026-08-21T04:12:33.128Z",
    "uploadedBy": "USR-001"
}
```

| Property | Selalu ada | Keterangan |
|----------|:----------:|------------|
| `key` | Ya | Identitas file di storage, dipakai untuk delete dan signed URL |
| `originalName` | Ya | Nama file asli dari client |
| `fileName` | Ya | Nama file setelah penambahan UUID |
| `contentType` | Ya | MIME type hasil deteksi saat upload |
| `size` | Ya | Ukuran file dalam bytes |
| `url` | Ya | URL akses file sesuai provider |
| `provider` | Ya | `s3` atau `local` |
| `uploadedAt` | Ya | Waktu upload dalam format ISO 8601 |
| `bucket` | Tidak | Hanya ditulis pada provider yang memiliki bucket |
| `uploadedBy` | Tidak | Diambil dari header `x-user-id` bila client mengirimkannya |

Field `key` adalah satu-satunya nilai yang dipakai runtime untuk operasi lanjutan. Property lain bersifat informatif dan boleh ditampilkan langsung di UI.

---

## Penyimpanan di Database

Array metadata disimpan pada satu kolom bertipe `json`. Runtime menangani perbedaan dialect secara otomatis:

| Dialect | Tipe kolom | Penanganan runtime |
|---------|------------|--------------------|
| PostgreSQL | JSONB | Array diserialisasi lalu di-cast ke `jsonb` saat INSERT dan UPDATE |
| MySQL | JSON | Array diserialisasi menjadi string JSON |
| Oracle | CLOB | Array diserialisasi menjadi string JSON |
| SQLite | TEXT | Array diserialisasi menjadi string JSON |

Pada operasi tulis, runtime yang menyerialisasi array. Pada operasi baca, nilai dikembalikan apa adanya sesuai driver dialect: dialect dengan tipe JSON native mengembalikan array object, sedangkan dialect yang menyimpan JSON sebagai teks (Oracle dan SQLite) mengembalikan string JSON yang perlu di-parse di sisi client.

---

## Alur Presigned URL

Presigned URL memungkinkan file dikirim langsung dari browser ke S3 tanpa melewati server aplikasi. Alur ini aktif hanya bila `uploadConfig.presignedUrl` bernilai `true` dan provider mendukungnya.

```
1. GET  /upload-url?field=photos&filename=foto.jpg   → server mengembalikan uploadUrl dan key
2. PUT  {uploadUrl}                                  → browser mengirim file langsung ke S3
3. POST /upload-confirm                              → server memverifikasi file lalu menyusun metadata
4. POST /create atau /update                         → metadata disimpan ke record
```

Perbedaan dengan alur biasa hanya pada dua langkah pertama. Mulai langkah 3, metadata yang dihasilkan identik dengan hasil `/upload`, kecuali `originalName` yang diturunkan dari key karena nama asli tidak ikut terkirim ke S3.

Provider `local` tidak mendukung presigned URL, sehingga permintaan ke `/upload-url` dijawab HTTP 501.

---

## File Orphan

File yang sudah masuk storage tetapi metadatanya tidak pernah disimpan akan menjadi orphan. Kondisi yang lazim menyebabkannya:

- User membatalkan form setelah memilih file.
- Browser tertutup atau koneksi terputus sebelum `/create` dipanggil.
- Validasi di endpoint CRUD gagal sehingga record tidak jadi tersimpan.

Runtime mencatat setiap key hasil `/upload` sebagai pending, lalu mengeluarkannya begitu `/create` atau `/update` berhasil. Key yang masih pending melewati ambang umur tertentu dihapus oleh cleanup job. Pengaturan jadwal dan ambang umur ada di [Konfigurasi Storage](configuration.md#cleanup-file-orphan).

Karena pencatatan pending memakai Redis, project tanpa Redis tidak memiliki proteksi orphan otomatis. Pada kondisi tersebut, pembersihan perlu ditangani di sisi bucket, misalnya lewat lifecycle rule.
