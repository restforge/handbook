# Field Upload Config (`uploadConfig`)

> Konfigurasi field penerima file upload beserta batasan tipe, ukuran, dan jumlah file.

Blok `uploadConfig` menentukan field mana pada resource yang menerima file, batasan apa yang berlaku, dan apakah presigned URL diaktifkan. Perilaku runtime, konfigurasi storage, serta alur panggilan endpoint dijelaskan di [`features/file-storage/`](../../features/file-storage/README.md).

---

## Prasyarat

| Prasyarat | Keterangan |
|-----------|------------|
| Kolom database | Field penerima file harus berupa kolom bertipe `json` di SDF (JSONB, JSON, CLOB, atau TEXT tergantung dialect) |
| Daftar field | Nama field wajib tercantum di array `fieldName` |
| Action | `action.upload` harus bernilai `true` |

Generator menolak RDF yang mengaktifkan `action.upload` tanpa blok `uploadConfig`. Registrasi route sendiri bertumpu pada keberadaan `uploadConfig.fields` di module hasil generate, sehingga blok yang ditulis tanpa `action.upload` tetap memunculkan endpoint upload. Menyelaraskan keduanya menjaga RDF tetap mencerminkan endpoint yang sebenarnya aktif.

`restforge payload generate` tidak pernah menulis `uploadConfig` secara otomatis, karena tidak ada penanda di struktur database yang menyatakan sebuah kolom JSON dipakai untuk metadata file. Blok ini ditulis manual, lalu endpoint di-generate ulang dengan `restforge endpoint create`.

---

## Struktur

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
            "allowedMimeTypes": ["image/jpeg", "image/png", "image/webp"],
            "storagePrefix": "photos"
        },
        "documents": {
            "columnType": "jsonb",
            "maxFiles": 5,
            "maxFileSize": "20MB",
            "allowedTypes": ["pdf"],
            "storagePrefix": "docs"
        }
    },
    "presignedUrl": false
}
```

Satu resource boleh memiliki lebih dari satu field upload, masing-masing dengan batasan sendiri.

---

## Property Level Blok

| Property | Tipe | Wajib | Default | Keterangan |
|----------|------|:-----:|---------|------------|
| `fields` | object | **Ya** | tidak ada | Peta nama field ke konfigurasi upload. Minimal satu entry, karena blok tanpa entry diabaikan saat registrasi endpoint |
| `presignedUrl` | boolean | Tidak | `false` | Bernilai `true` mendaftarkan endpoint `/upload-url` dan `/upload-confirm` untuk upload langsung dari client ke storage |

---

## Property Per Field

| Property | Tipe | Wajib | Default | Keterangan |
|----------|------|:-----:|---------|------------|
| `columnType` | string | Tidak | tidak ada | Tipe kolom penampung metadata: `jsonb`, `json`, `clob`, atau `text`. Bersifat dokumentatif, dan nilai di luar daftar hanya memunculkan peringatan saat validasi |
| `maxFiles` | number | Tidak | `5` | Jumlah maksimum file per request upload. Harus bilangan positif |
| `maxFileSize` | string | Tidak | `"10MB"` | Ukuran maksimum per file. Format angka diikuti satuan `B`, `KB`, `MB`, atau `GB`, misalnya `"500KB"` atau `"1.5GB"` |
| `allowedTypes` | array of string | Tidak | seluruh extension | Daftar extension yang diizinkan, ditulis tanpa titik dan dalam huruf kecil. Array kosong ditolak saat validasi |
| `allowedMimeTypes` | array of string | Tidak | seluruh MIME type | Daftar MIME type yang diizinkan, diperiksa sebagai lapis tambahan setelah extension |
| `storagePrefix` | string | Tidak | tidak ada | Segmen tambahan pada storage key untuk memisahkan file antar field |

Batasan `maxFiles` berlaku per request. Jumlah kumulatif file yang tersimpan pada satu record tidak dibatasi runtime, sehingga pembatasan tersebut perlu ditegakkan di sisi client atau lewat komponen validasi tersendiri.

---

## Aturan Validasi

Validasi berikut dijalankan saat `restforge payload validate` maupun saat generate endpoint.

| Kondisi | Hasil |
|---------|-------|
| `action.upload` aktif tanpa `uploadConfig` | Error |
| `uploadConfig` bukan object | Error |
| `uploadConfig.fields` tidak ada atau bukan object | Error |
| Nama field tidak tercantum di `fieldName` | Error |
| `maxFiles` bukan number atau kurang dari 1 | Error |
| `maxFileSize` tidak mengikuti pola angka dan satuan | Error |
| `allowedTypes` bukan array atau berupa array kosong | Error |
| `columnType` di luar `jsonb`, `json`, `clob`, `text` | Peringatan, generate tetap berjalan |

---

## Endpoint yang Dihasilkan

| Endpoint | Terdaftar saat | Permission auth guard |
|----------|----------------|-----------------------|
| `POST /upload` | Selalu, bila blok `fields` berisi minimal satu field | `UPDATE` |
| `POST /upload-delete` | Selalu | `DELETE` |
| `GET /upload-url` | `presignedUrl: true` | `UPDATE` |
| `POST /upload-confirm` | `presignedUrl: true` | `UPDATE` |
| `POST /upload-cleanup` | Selalu | `DELETE` |

Kontrak request dan response setiap endpoint ada di [`api-spec/endpoint-upload.md`](../../api-spec/endpoint-upload.md). Pemetaan permission berlaku ketika resource memakai [`authGuard`](./auth-guard.md).

---

## Contoh Lengkap

```json
{
    "tableName": "upload_item",
    "primaryKey": "upload_id",
    "fieldName": ["upload_id", "name", "photos", "created_at", "created_by", "updated_at", "updated_by"],
    "fieldValidation": [
        { "name": "upload_id", "type": "string", "constraints": { "primaryKey": true, "required": true } },
        { "name": "name", "type": "string", "constraints": { "required": true } }
    ],
    "datatablesQuery": "select upload_id, name, photos from upload_item",
    "datatablesWhere": ["name", "all"],
    "action": {
        "datatables": true,
        "create": true,
        "read": true,
        "update": true,
        "delete": true,
        "first": true,
        "upload": true
    },
    "uploadConfig": {
        "fields": {
            "photos": {
                "columnType": "jsonb",
                "maxFiles": 10,
                "maxFileSize": "5MB",
                "allowedTypes": ["jpg", "jpeg", "png", "gif", "webp"],
                "storagePrefix": "photos"
            }
        },
        "presignedUrl": false
    }
}
```

---

**Lihat juga**: [`features/file-storage/`](../../features/file-storage/README.md) · [`api-spec/endpoint-upload.md`](../../api-spec/endpoint-upload.md) · [`README`](./README.md)
