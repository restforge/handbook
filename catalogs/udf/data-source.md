# Sumber Data — `dataSource`

> Konfigurasi sumber data untuk dropdown `select` dan filter `dataFilters[]`.

## Dua Tipe Sumber Data

Generator mendukung dua tipe sumber data:

| Tipe | `dataSource.type` | Cocok untuk |
|------|-------------------|-------------|
| Static | `"static"` | Data tetap, jumlah sedikit (<10 opsi) |
| API | `"api"` | Data dinamis, jumlah banyak, sering berubah |

Struktur `dataSource` yang sama dipakai di field bertipe `select` dan di `dataFilters[]` (filter di toolbar).

## Static Source

Opsi dropdown didefinisikan langsung di payload sebagai array.

```json
{
    "name": "contact_type",
    "label": "Tipe Kontak",
    "type": "select",
    "required": true,
    "dataSource": {
        "type": "static",
        "options": [
            {"value": "personal", "text": "Personal"},
            {"value": "business", "text": "Business"},
            {"value": "family", "text": "Family"}
        ]
    }
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `type` | string | ✓ | Harus bernilai `"static"` |
| `options` | array | ✓ | Array objek `{value, text}` |
| `options[].value` | string | ✓ | Nilai yang disimpan ke database |
| `options[].text` | string | ✓ | Teks yang ditampilkan di dropdown |

### Perilaku Select Required vs Optional

| Kondisi | Behavior |
|---------|----------|
| `required: true` | Opsi pertama langsung terpilih, tanpa placeholder |
| `required: false` (default) | Menampilkan placeholder `-- Select --` dengan value kosong sebagai opsi pertama |

## API Source

Opsi dropdown dimuat secara dinamis dari endpoint backend lewat AJAX saat halaman dimuat.

```json
{
    "name": "city_id",
    "label": "Kota",
    "type": "select",
    "inTable": true,
    "tableOrder": 4,
    "tableField": "city_name",
    "dataSource": {
        "type": "api",
        "resource": "city",
        "select": ["city_id", "city_name"]
    }
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `type` | string | ✓ | Harus bernilai `"api"` |
| `resource` | string | ✓ | Nama resource backend. Dipakai di pola endpoint `POST /<apiBaseUrl>/<resource>/lookup` |
| `select` | array of string | ✗ | Nama kolom yang di-request ke API lookup |

### Mekanisme Kerja Select API

Saat halaman dimuat, JavaScript yang di-generate menjalankan urutan berikut:

1. Kirim `POST` ke endpoint `<apiBaseUrl>/<resource>/lookup`
2. Sertakan header `x-request-mode: static` dan body `{ "select": [...] }`
3. Tunggu response dengan format `{ "success": true, "data": [{"id": "...", "text": "..."}, ...] }`
4. Render setiap objek sebagai `<option>` dan inisialisasi sebagai Select2

### `tableField` untuk Foreign Key

Atribut `tableField` diperlukan ketika field di form (`city_id`) berbeda dengan kolom display di DataTable (`city_name`):

```
Form save/load:   city_id   → "ct-00000-0004-..."
DataTable render: city_name → "Bandung"
```

Tanpa `tableField`, DataTable mencari `city_id` di response yang berisi UUID, bukan nama kota.

## `dataSource` di `dataFilters[]`

Filter di toolbar (`features.enableDataFilter: true`) memakai struktur `dataSource` yang sama dengan field `select`:

```json
{
    "features": {
        "enableDataFilter": true,
        "dataFilters": [
            {
                "name": "city_id",
                "label": "Kota",
                "dataSource": {
                    "type": "api",
                    "resource": "city",
                    "select": ["city_id", "city_name"]
                }
            },
            {
                "name": "contact_type",
                "label": "Tipe Kontak",
                "dataSource": {
                    "type": "static",
                    "options": [
                        {"value": "personal", "text": "Personal"},
                        {"value": "business", "text": "Business"}
                    ]
                }
            }
        ]
    }
}
```

Lihat [`features.md`](./features.md) untuk detail `enableDataFilter`.

## Perbandingan Static vs API

| Aspek | Static | API |
|-------|--------|-----|
| Sumber data | Didefinisikan di payload | Dimuat dari endpoint `/<resource>/lookup` |
| Waktu load | Tersedia langsung saat halaman dibuka | Memerlukan HTTP POST saat halaman dimuat |
| Cocok untuk | Data tetap, jumlah sedikit (<10 opsi) | Data dinamis, jumlah banyak, sering berubah |
| Dependensi | Tidak memerlukan backend | Memerlukan backend yang berjalan |
| `tableField` | Tidak diperlukan | Diperlukan jika display value berbeda dari foreign key |

## Aturan Validasi

| Aturan | Hasil |
|--------|-------|
| Field `select` tanpa `dataSource` | Warning: `"Field '<name>' is of type 'select' but has no dataSource"` |
| `dataSource.type` selain `"static"` / `"api"` | Tidak divalidasi formal oleh validator umum, tetapi plugin runtime mengabaikan dataSource yang tidak dikenali |

> **Catatan:** Validator UDF tidak memvalidasi struktur internal `dataSource` (mis. apakah `options[].value` ada). Validasi tingkat plugin atau runtime menangani sisanya. Jika ada plugin custom yang menambah tipe `dataSource` baru, dokumentasi plugin tersebut yang menjadi referensi.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Daftar provinsi Indonesia (34 opsi) | `static` karena data jarang berubah |
| Daftar kota di Indonesia (500+ opsi) | `api` karena data dinamis dan banyak |
| Status pesanan (4 opsi: Draft, Submitted, Approved, Rejected) | `static` |
| Daftar customer (ratusan, bisa bertambah) | `api` |
| Daftar tipe transaksi (5 opsi tetap) | `static` |

---

← [`field-attributes.md`](./field-attributes.md) | [Selanjutnya: `id-generation.md`](./id-generation.md) →
