# Konfigurasi Storage

> Pemilihan provider, variable `.env`, format storage key, dan cleanup file orphan.

Kembali ke [ikhtisar File Storage](README.md).

---

## Pemilihan Provider

Provider ditentukan oleh `STORAGE_PROVIDER` dan bersifat satu instance per proses server. Nilai yang dikenali:

| Nilai | Provider | Penggunaan |
|-------|----------|------------|
| `s3` atau `aws` | AWS S3 dan S3-compatible | Produksi |
| `local` | Local filesystem | Development, sekaligus nilai default bila variable tidak diisi |

Provider bersifat sistem, bukan per resource. Seluruh resource pada satu server memakai provider yang sama.

---

## Variable AWS S3

```env
# ===========================================
# STORAGE SYSTEM
# ===========================================
STORAGE_ENABLED=true
STORAGE_PROVIDER=s3

# AWS S3
S3_BUCKET=my-bucket
S3_REGION=ap-southeast-1
S3_ACCESS_KEY_ID=AKIA...
S3_SECRET_ACCESS_KEY=xxxxxxxx

# S3-compatible (MinIO, DigitalOcean Spaces)
# S3_ENDPOINT=http://localhost:9000
# S3_FORCE_PATH_STYLE=true
```

| Variable | Default | Keterangan |
|----------|---------|------------|
| `STORAGE_ENABLED` | tidak diisi | Bernilai `true` mengaktifkan subsistem storage saat startup, termasuk cleanup job terjadwal |
| `STORAGE_PROVIDER` | `local` | Pemilihan provider (`s3`, `aws`, atau `local`) |
| `S3_BUCKET` | tidak ada | Nama bucket. Wajib diisi saat provider `s3`, karena provider menolak inisialisasi tanpa nilai ini |
| `S3_REGION` | `ap-southeast-1` | Region bucket |
| `S3_ACCESS_KEY_ID` | tidak diisi | Access key. Bila dikosongkan bersama secret key, AWS SDK memakai credential chain bawaannya (IAM role, profile, atau variable standar AWS) |
| `S3_SECRET_ACCESS_KEY` | tidak diisi | Secret key, dipakai berpasangan dengan access key |
| `S3_ENDPOINT` | tidak diisi | Endpoint kustom untuk layanan S3-compatible seperti MinIO dan DigitalOcean Spaces |
| `S3_FORCE_PATH_STYLE` | `false` | Bernilai `true` memaksa format path-style (`{endpoint}/{bucket}/{key}`), yang dibutuhkan MinIO |

Endpoint upload sendiri terdaftar berdasarkan blok `uploadConfig` pada resource, sehingga tetap aktif walaupun `STORAGE_ENABLED` tidak diisi. Yang bergantung pada variable tersebut adalah pendaftaran cleanup job terjadwal saat startup. Tanpa `STORAGE_PROVIDER=s3`, file yang masuk lewat endpoint upload disimpan di local filesystem.

### Bentuk URL File

URL yang tersimpan pada metadata mengikuti konfigurasi di atas:

| Kondisi | Bentuk URL |
|---------|-----------|
| AWS S3 standar | `https://{bucket}.s3.{region}.amazonaws.com/{key}` |
| `S3_ENDPOINT` diisi, path-style aktif | `{endpoint}/{bucket}/{key}` |
| `S3_ENDPOINT` diisi, path-style nonaktif | `{endpoint}/{key}` |

URL ini disimpan apa adanya di database. Bila bucket bersifat privat, akses file dilakukan lewat signed URL, bukan lewat URL tersebut.

### Perilaku Upload ke S3

| Aspek | Perilaku |
|-------|----------|
| Metode | Multipart upload, sehingga file besar dipecah menjadi beberapa part |
| Ukuran part | 5 MB |
| Paralelisme | 4 part sekaligus |
| File kecil | SDK otomatis memakai single PUT, tanpa overhead multipart |
| Kegagalan di tengah proses | Part yang sudah terkirim dibatalkan dan dibersihkan, sehingga tidak menyisakan potongan file |
| Batch delete | Sampai 1000 object per request, dipecah otomatis bila lebih |

---

## Variable Local Filesystem

Provider `local` ditujukan untuk development dan tidak mendukung presigned URL.

| Variable | Default | Keterangan |
|----------|---------|------------|
| `UPLOAD_STORAGE_DIR` | `{cwd}/uploads/{project}` | Folder penyimpanan file |
| `UPLOAD_BASE_URL` | `/uploads/{project}` | Prefix URL yang ditulis pada metadata |

Nilai `{project}` diambil dari `RESTFORGE_PROJECT_NAME`, dan menjadi `default` bila variable tersebut tidak diisi. URL yang dihasilkan berupa path relatif, sehingga file perlu disajikan sendiri bila ingin diakses langsung dari browser.

---

## Variable Proses Upload

| Variable | Default | Keterangan |
|----------|---------|------------|
| `UPLOAD_TEMP_DIR` | `{temp OS}/restforge-uploads/{project}` | Folder penampungan sementara sebelum file dikirim ke storage. File temp dihapus setelah upload selesai maupun gagal |
| `UPLOAD_PRESIGNED_EXPIRY` | `3600` | Masa berlaku presigned URL dalam detik |

Lokasi default folder temp sengaja berada di luar folder project agar penulisan file sementara tidak memicu restart pada process manager yang memantau perubahan file.

---

## Format Storage Key

Setiap file mendapat key unik yang dibentuk otomatis:

```
{project}/{module}/{endpoint}/{storagePrefix}/{YYYY-MM}/{uuid}-{nama-file}.{ext}
```

Contoh: `mini-inventory/mini-inventory/upload-item/photos/2026-08/0191f3c2-1b7e-7c31-9c2f-2d5b8a1e4f60-foto-produk.jpg`

| Segmen | Sumber |
|--------|--------|
| `project` | `RESTFORGE_PROJECT_NAME`, atau `default` |
| `module` dan `endpoint` | Nama module dan resource tempat file di-upload |
| `storagePrefix` | Property `uploadConfig.fields[field].storagePrefix`, dilewati bila tidak diisi |
| `YYYY-MM` | Tahun dan bulan saat upload |
| `uuid` | UUID v7 yang menjamin keunikan sekaligus urut secara waktu |
| `nama-file` | Nama asli yang disanitasi, karakter selain huruf, angka, garis bawah, dan tanda hubung diganti tanda hubung, lalu dipotong maksimal 50 karakter |

Segmen yang kosong dilewati, jadi key tidak pernah memuat garis miring ganda.

---

## Cleanup File Orphan

File orphan adalah file yang sudah terkirim ke storage namun metadatanya tidak pernah disimpan ke database, misalnya karena user membatalkan form atau koneksi terputus sebelum `/create` dipanggil.

Pencatatannya memakai Redis: setiap key hasil `/upload` masuk ke daftar pending, lalu dikeluarkan dari daftar tersebut begitu `/create` atau `/update` berhasil menyimpan metadata. Tanpa Redis, upload tetap berjalan normal dan hanya pencatatan orphan yang tidak aktif.

| Variable | Default | Keterangan |
|----------|---------|------------|
| `UPLOAD_CLEANUP_CRON` | `0 */6 * * *` | Jadwal cleanup, secara default setiap 6 jam |
| `UPLOAD_CLEANUP_THRESHOLD` | `86400` | Umur minimal file pending dalam detik sebelum dianggap orphan, secara default 24 jam |

Job terjadwal hanya didaftarkan saat startup apabila `STORAGE_ENABLED=true`. Koneksi Redis untuk job ini memakai variable Redis yang berlaku di project (`REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_DB`). Pembersihan yang sama dapat dipicu manual kapan saja lewat endpoint [`/upload-cleanup`](../../api-spec/endpoint-upload.md#cleanup-orphan-post-upload-cleanup).

Key yang gagal dihapus dari storage tetap berada di daftar pending, sehingga akan dicoba lagi pada siklus cleanup berikutnya.

---

## Hierarki Konfigurasi

| Lapis | Berlaku untuk | Ditentukan di |
|-------|---------------|---------------|
| Sistem | Provider, kredensial, bucket, jadwal cleanup | `.env` atau `config/*.env` |
| Resource | Field mana yang menerima file, batas ukuran dan tipe, presigned URL | `uploadConfig` pada RDF |
| Request | Field tujuan dan file yang dikirim | Parameter request ke endpoint upload |

Batasan pada level resource tidak dapat dilonggarkan dari request. Bila file melanggar `maxFileSize` atau `allowedTypes`, request ditolak dengan HTTP 400 dan file tidak pernah sampai ke storage.
