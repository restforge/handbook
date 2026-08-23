# File Storage

> Penyimpanan file binary ke AWS S3 (atau S3-compatible) dengan metadata file tersimpan di kolom JSON pada tabel resource.

Fitur File Storage menambahkan sekumpulan endpoint upload pada resource yang mendeklarasikan `uploadConfig`. File binary dikirim ke object storage, sedangkan yang masuk ke database hanyalah metadata file berupa array JSON. Pemisahan ini disengaja: endpoint upload tidak pernah menyentuh database, dan endpoint CRUD tidak pernah menyentuh storage.

Tiga bagian yang membentuk fitur ini:

- **Storage provider** — lapisan abstraksi dengan dua implementasi, yaitu S3 untuk produksi dan local filesystem untuk development.
- **Endpoint upload** — `/upload`, `/upload-delete`, `/upload-url`, `/upload-confirm`, dan `/upload-cleanup` yang terdaftar otomatis per resource.
- **Siklus metadata** — pencatatan file pending, konfirmasi saat metadata tersimpan lewat `/create` atau `/update`, cascade delete saat record dihapus, dan pembersihan file orphan terjadwal.

---

## Referensi Cepat

| Properti | Nilai |
|----------|-------|
| Database | PostgreSQL, MySQL, Oracle, SQLite |
| Storage provider | `s3` (AWS S3 dan S3-compatible), `local` (default) |
| Dependency | Sudah termasuk dalam `@restforgejs/platform` (AWS SDK v3 dan multer) |
| Dependency opsional | Redis, untuk tracking file orphan dan cleanup terjadwal |
| Aktivasi RDF | `action.upload: true` dan blok `uploadConfig` |
| Tipe kolom metadata | Kolom bertipe `json` di SDF (JSONB, JSON, CLOB, atau TEXT sesuai dialect) |
| Endpoint | `/upload`, `/upload-delete`, `/upload-url`, `/upload-confirm`, `/upload-cleanup` |
| Konfigurasi storage | `STORAGE_PROVIDER`, `S3_BUCKET`, `S3_REGION`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_ENDPOINT`, `S3_FORCE_PATH_STYLE` |
| Konfigurasi upload | `STORAGE_ENABLED`, `UPLOAD_TEMP_DIR`, `UPLOAD_PRESIGNED_EXPIRY`, `UPLOAD_CLEANUP_CRON`, `UPLOAD_CLEANUP_THRESHOLD` |

---

## Setup Singkat

Tiga langkah untuk mengaktifkan upload file ke AWS S3 pada satu resource.

### 1. Siapkan kolom metadata di SDF

Metadata file disimpan sebagai array JSON pada satu kolom. Deklarasikan kolom tersebut bertipe `json` di file schema, lalu jalankan migrasi:

```javascript
// schema/upload-item.js
module.exports = ({ defineModel }) => defineModel('upload_item', {
  fields: {
    upload_id:  'uuid pk',
    name:       'string:100 notnull',
    photos:     'json',
    created_at: 'timestamp',
    created_by: 'string:70',
    updated_at: 'timestamp',
    updated_by: 'string:70'
  }
});
```

Tipe `json` diterjemahkan menjadi JSONB pada PostgreSQL, JSON pada MySQL, CLOB pada Oracle, dan TEXT pada SQLite.

### 2. Deklarasikan `uploadConfig` di RDF

Aktifkan action `upload` dan daftarkan field file beserta batasannya, lalu generate ulang endpoint dengan `restforge endpoint create`:

```json
"action": {
    "datatables": true,
    "create": true,
    "update": true,
    "delete": true,
    "upload": true
},
"uploadConfig": {
    "fields": {
        "photos": {
            "columnType": "jsonb",
            "maxFiles": 10,
            "maxFileSize": "5MB",
            "allowedTypes": ["jpg", "jpeg", "png", "webp"],
            "storagePrefix": "photos"
        }
    },
    "presignedUrl": false
}
```

Daftar lengkap property ada di [`catalogs/rdf/upload-config.md`](../../catalogs/rdf/upload-config.md).

### 3. Arahkan storage ke bucket S3

Tambahkan variable berikut di file `.env` (root) atau `config/*.env`, lalu restart server:

```env
STORAGE_ENABLED=true
STORAGE_PROVIDER=s3
S3_BUCKET=my-bucket
S3_REGION=ap-southeast-1
S3_ACCESS_KEY_ID=AKIA...
S3_SECRET_ACCESS_KEY=xxxxxxxx
```

Setelah restart, resource tersebut memiliki endpoint upload aktif. Tanpa `STORAGE_PROVIDER=s3`, file tetap dapat di-upload namun tersimpan di local filesystem. Detail seluruh variable ada di [Konfigurasi Storage](configuration.md).

---

## Batasan Tanggung Jawab

| Operasi | Ditangani oleh | Menyentuh storage | Menyentuh database |
|---------|----------------|:-----------------:|:------------------:|
| Kirim file binary | `/upload` | Ya | Tidak |
| Simpan metadata file pada record | `/create` atau `/update` | Tidak | Ya |
| Hapus file dari storage | `/upload-delete` | Ya | Tidak |
| Hapus metadata dari record | `/update` | Tidak | Ya |
| Hapus record beserta filenya | `/delete` | Ya, lewat cascade otomatis | Ya |

Konsekuensinya, client wajib melakukan dua panggilan untuk satu perubahan file: satu ke endpoint upload dan satu ke endpoint CRUD. Alasan pemisahan serta pola panggilannya dijelaskan di [Alur Kerja dan Metadata](workflow.md).

---

## Dokumentasi Lengkap

| Dokumen | Isi |
|---------|-----|
| [Konfigurasi Storage](configuration.md) | Pemilihan provider, seluruh variable `.env`, format storage key, cleanup terjadwal |
| [Alur Kerja dan Metadata](workflow.md) | Alur create, update, delete, struktur metadata file, file orphan, cascade delete |
| [`uploadConfig`](../../catalogs/rdf/upload-config.md) | Spec property RDF untuk mengaktifkan upload per field |
| [Endpoint Upload](../../api-spec/endpoint-upload.md) | Kontrak request dan response kelima endpoint upload |
