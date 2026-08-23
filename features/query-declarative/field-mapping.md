# Kolom Response dan Validasi Input (Field Mapping and Validation)

> Peran `fieldName` dan `fieldValidation` terhadap query, response, dan operasi write.

Kembali ke [ikhtisar Query Declarative](README.md).

`fieldName` menentukan kolom yang dikembalikan ke client, sedangkan `fieldValidation` menentukan validasi dan sanitasi saat write. Keduanya bekerja di level aplikasi dan tidak mengubah SQL query yang dieksekusi. Spec constraint lengkap ada di referensi [`catalogs/rdf/field-validation.md`](../../catalogs/rdf/field-validation.md).

---

## Mengatur Kolom Response dengan fieldName (Response Columns)

`fieldName` adalah whitelist kolom yang dikenali module. Setiap hasil query difilter berdasarkan daftar ini sebelum dikirim ke client.

```json
{
  "fieldName": [
    "item_product_id", "product_code", "sku", "product_name",
    "category_id", "category_name", "selling_price", "stock",
    "uom", "is_active"
  ]
}
```

Kolom di luar `fieldName` dibuang dari response. Mekanisme filtering ini berlaku untuk `/datatables`, `/read`, `/first`, dan `/lookup`.

```
Database mengembalikan 10 kolom
        ↓
Filter response berdasarkan fieldName (7 kolom)
        ↓
Client menerima 7 kolom (3 kolom ekstra dibuang)
```

Selain memfilter response, `fieldName` juga membatasi kolom yang boleh dipakai pada operasi lain.

| Operasi | Fungsi `fieldName` |
|---------|--------------------|
| SELECT (response) | Memfilter kolom yang dikembalikan ke client |
| INSERT | Hanya kolom terdaftar yang masuk ke statement INSERT |
| UPDATE | Hanya kolom terdaftar yang masuk ke statement UPDATE |
| WHERE | Hanya kolom terdaftar yang boleh dipakai sebagai filter |
| ORDER BY | Hanya kolom terdaftar yang boleh dipakai sebagai pengurutan |

---

## Mendaftarkan Kolom Relasi (Registering Join Columns)

Kolom hasil JOIN harus didaftarkan di `fieldName` agar muncul di response. Tanpa pendaftaran, kolom tetap di-query dari database tetapi dibuang sebelum dikirim ke client.

```json
{
  "fieldName": [
    "item_product_id", "product_code", "product_name",
    "category_id", "category_name"
  ],
  "datatablesQuery": "select a.item_product_id, a.product_code, a.product_name, a.category_id, b.category_name from item_product a inner join category b on b.category_id = a.category_id"
}
```

| Skenario | Perilaku |
|----------|----------|
| Query mengembalikan kolom lebih banyak dari `fieldName` | Kolom ekstra dibuang, berguna mencegah kolom internal bocor |
| `fieldName` memuat kolom yang tidak ada di hasil query | Kolom tersebut tidak muncul, tanpa error |
| Kolom JOIN tidak didaftarkan di `fieldName` | Kolom JOIN dibuang dari response |

> **Catatan:** `fieldName` tidak mengubah SQL yang dieksekusi. Query `datatablesQuery`, `viewQuery`, dan `exportQuery` tetap berjalan apa adanya, dan filtering terjadi setelah query dieksekusi.

---

## Memvalidasi Input dengan fieldValidation (Input Validation)

`fieldValidation` mendefinisikan tipe data dan constraint untuk operasi write (INSERT dan UPDATE). Konfigurasi ini bekerja saat data masuk, bukan saat query SELECT.

```json
{
  "fieldValidation": [
    {
      "name": "product_code",
      "type": "string",
      "constraints": {
        "required": true,
        "maxLength": 20,
        "unique": true,
        "trim": true,
        "uppercase": true
      }
    },
    {
      "name": "selling_price",
      "type": "number",
      "constraints": { "required": false, "min": 0, "default": 0 }
    }
  ]
}
```

Alur write menjalankan validasi lalu sanitasi sebelum data masuk ke database.

```
Request data masuk (POST /add atau POST /update)
        ↓
Validasi: required, maxLength, min/max, format
        ↓
Sanitasi: trim, uppercase, default
        ↓
Hanya field di fieldName yang masuk ke SQL
```

> **Catatan:** `fieldValidation` tidak memengaruhi query SELECT apa pun. Tidak ada hubungan langsung dengan `datatablesQuery`, `viewQuery`, atau `exportQuery`. Daftar tipe dan constraint yang didukung ada di referensi [`catalogs/rdf/field-validation.md`](../../catalogs/rdf/field-validation.md).

---

## Ringkasan Peran (Role Summary)

| Komponen | Memengaruhi SQL | Memengaruhi Response | Memengaruhi Input |
|----------|:----------------:|:---------------------:|:------------------:|
| `fieldName` | Tidak | Ya, filter kolom output | Ya, filter kolom INSERT/UPDATE |
| `fieldValidation` | Tidak | Tidak | Ya, validasi dan sanitasi input |
| Query declarations | Ya, menentukan SQL | Tidak langsung | Tidak |

---

**Lihat juga**: [Field Validation (RDF)](../../catalogs/rdf/field-validation.md) · [Query List dan Read](read-queries.md) · [Field Sumber Data (RDF)](../../catalogs/rdf/data-source.md)
