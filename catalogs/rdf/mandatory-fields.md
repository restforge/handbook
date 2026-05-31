# Field Wajib (Mandatory Fields)

## `tableName`

Nama tabel database tempat resource ini di-map. Wajib snake_case, sesuai konvensi DDL RESTForge.

```json
{ "tableName": "category" }
```

Untuk dialect yang mendukung schema namespace (PostgreSQL, Oracle, SQL Server), tabel dapat di-prefix dengan nama schema:

```json
{ "tableName": "inventory.item_product" }
```

| Aturan | Penjelasan |
|--------|-----------|
| Format | `<table>` atau `<schema>.<table>` |
| Case | snake_case |
| Wajib | Ya |
| Validasi runtime | Tabel harus exist di database saat module di-load |

## `primaryKey`

Nama kolom primary key. Digunakan oleh action `first`, `update`, `delete`, dan untuk auto-detect saat operasi composite.

```json
{ "primaryKey": "category_id" }
```

| Aturan | Penjelasan |
|--------|-----------|
| Default | `fieldName[0]` jika tidak di-set |
| Format | snake_case, harus exist di `fieldName` |
| Wajib | Tidak (tapi sangat direkomendasikan untuk eksplisit) |
| Composite PK | Tidak didukung di RDF, gunakan single-column PK |

## `fieldName`

Array nama kolom yang akan di-expose sebagai API field. Urutan signifikan dan menentukan prioritas tampilan di response.

```json
{
    "fieldName": [
        "category_id",
        "category_code",
        "category_name",
        "description",
        "is_active"
    ]
}
```

| Aturan | Penjelasan |
|--------|-----------|
| Wajib | Ya |
| Minimal | 1 item |
| Item | Nama kolom database (snake_case) |
| Urutan | Menentukan urutan kolom di response `/datatables`, `/read`, `/first` |
| Hubungan dengan `fieldValidation` | Setiap field di `fieldValidation` harus exist di `fieldName` |

## `action`

Object yang berisi flag boolean per action. Setiap action yang bernilai `true` akan menghasilkan endpoint di runtime.

```json
{
    "action": {
        "datatables": true,
        "create": true,
        "update": true,
        "delete": true,
        "first": true,
        "lookup": true,
        "read": true
    }
}
```

### Daftar Action

| Action | Endpoint | Method | Tujuan |
|--------|----------|--------|--------|
| `datatables` | `/{endpoint}/datatables` | POST | List data dengan pagination + search |
| `create` | `/{endpoint}/create` | POST | Insert record baru |
| `update` | `/{endpoint}/update` | POST | Update record berdasarkan primary key |
| `delete` | `/{endpoint}/delete` | POST | Hapus record berdasarkan primary key |
| `first` | `/{endpoint}/first` | POST | Ambil satu record berdasarkan primary key |
| `lookup` | `/{endpoint}/lookup` | GET, POST | Data dropdown/autocomplete |
| `read` | `/{endpoint}/read` | POST | Baca data dengan WHERE clause |
| `export` | `/{endpoint}/export` | POST | Export data ke Excel |
| `import` | `/{endpoint}/import-upload`, `/import-preview`, `/import-commit` | POST | Import data dari Excel (3 langkah) |
| `adjust` | `/{endpoint}/adjust` | POST | Increment/decrement field numerik atomic |
| `aggregate` | `/{endpoint}/aggregate` | POST | Operasi SUM, COUNT, AVG, MIN, MAX dengan GROUP BY |
| `createComposite` | `/{endpoint}/create-composite` | POST | Insert header + detail dalam satu transaksi |
| `updateComposite` | `/{endpoint}/update-composite` | POST | Update header + detail dalam satu transaksi |
| `readComposite` | `/{endpoint}/read-composite` | POST | Baca header + detail |
| `workflow` | `/{endpoint}/change-status` | POST | Transisi status dengan validasi |

| Aturan | Penjelasan |
|--------|-----------|
| Default | `false` jika action tidak ada di object |
| `additionalProperties` | Tidak diizinkan, action di luar daftar di atas akan throw error |
| Aktivasi parsial | Setiap action independen, bebas dikombinasikan |
| Dependency | `createComposite`/`updateComposite`/`readComposite` mensyaratkan `masterDetail` di-set. `adjust` mensyaratkan `adjustConfig`. `workflow` mensyaratkan `workflow` object dan kolom status di `fieldName` |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
