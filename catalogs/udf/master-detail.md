# Master Detail — `details[]`

> Pola halaman master-detail: satu page CRUD memiliki satu atau lebih tabel detail yang terkait.

## Konsep

Pola master-detail dipakai ketika satu record (master) memiliki kumpulan baris terkait (detail). Contoh klasik:

| Master | Detail |
|--------|--------|
| Sales Order | Order Item (banyak baris per order) |
| Invoice | Invoice Line Item |
| Stock Inbound | Stock Inbound Item |
| Purchase Order | PO Line Item |

UDF mendeklarasikan detail di field `details[]` di page object master. Setiap entry di `details[]` mendefinisikan satu tabel detail.

## Sintaks

```json
{
    "pageId": "sales-order",
    "pageTitle": "Sales Order",
    "primaryKey": "order_id",
    "displayField": "order_no",
    "apiPath": "sales-order",
    "fields": [
        {"name": "order_no", "label": "No Order", "type": "text", "required": true},
        {"name": "customer_id", "label": "Customer", "type": "select", "dataSource": {"type": "api", "resource": "customer"}}
    ],
    "details": [
        {
            "detailId": "items",
            "primaryKey": "item_id",
            "fields": [
                {"name": "product_id", "label": "Product", "type": "select", "dataSource": {"type": "api", "resource": "product"}},
                {"name": "qty", "label": "Qty", "type": "number"},
                {"name": "price", "label": "Price", "type": "number"}
            ]
        }
    ]
}
```

## Properti `details[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `detailId` | string | ✓ | Identifier detail (snake_case atau camelCase). Dipakai sebagai key di payload save dan URL endpoint detail |
| `primaryKey` | string | ✓ | Nama field primary key untuk baris detail |
| `fields` | array | ✓ | Array definisi field detail (minimal 1, sama struktur seperti field master) |

### Aturan Validasi `details[]`

| Aturan | Error |
|--------|-------|
| `detailId` wajib | `"[<pageId>].details[<i>] detailId must be provided"` |
| `primaryKey` wajib | `"[<pageId>].details[<detailId>] primaryKey must be provided"` |
| `fields` wajib non-empty | `"[<pageId>].details[<detailId>] fields must be provided and cannot be empty"` |
| Setiap field di `details[].fields[]` wajib `name`, `label`, `type` valid | Error per-field seperti di master |

## Field di Detail

Field di dalam `details[].fields[]` memakai struktur dan tipe yang sama dengan field master (lihat [`field-types.md`](./field-types.md) dan [`field-attributes.md`](./field-attributes.md)). Atribut umum (`required`, `inTable`, `tableOrder`, `placeholder`, `readonly`, dll.) berlaku penuh.

```json
"details": [
    {
        "detailId": "items",
        "primaryKey": "item_id",
        "fields": [
            {
                "name": "product_id",
                "label": "Produk",
                "type": "select",
                "required": true,
                "inTable": true,
                "tableOrder": 1,
                "tableField": "product_name",
                "dataSource": {
                    "type": "api",
                    "resource": "product",
                    "select": ["product_id", "product_name"]
                }
            },
            {
                "name": "qty",
                "label": "Qty",
                "type": "number",
                "required": true,
                "inTable": true,
                "tableOrder": 2,
                "min": 1
            },
            {
                "name": "price",
                "label": "Harga",
                "type": "number",
                "required": true,
                "inTable": true,
                "tableOrder": 3
            }
        ]
    }
]
```

## Multiple Details Per Master

Satu master dapat memiliki beberapa tabel detail:

```json
{
    "pageId": "purchase-order",
    "details": [
        {
            "detailId": "items",
            "primaryKey": "po_item_id",
            "fields": [ /* ... */ ]
        },
        {
            "detailId": "attachments",
            "primaryKey": "attachment_id",
            "fields": [ /* ... */ ]
        },
        {
            "detailId": "approvals",
            "primaryKey": "approval_id",
            "fields": [ /* ... */ ]
        }
    ]
}
```

Setiap detail di-render sebagai bagian terpisah di form master.

## Endpoint API yang Dipakai

Generator menggunakan pola endpoint berikut untuk operasi detail:

| Operasi | Endpoint |
|---------|----------|
| Load detail per master | `GET <apiBaseUrl>/<apiPath>/<masterId>/<detailId>` |
| Save master + details (composite) | `POST <apiBaseUrl>/<apiPath>` dengan body `{master: {...}, <detailId>: [...]}` |
| Update master + details | `PUT <apiBaseUrl>/<apiPath>/<masterId>` dengan body composite |

Detail komposisi payload save dan response struktur ditangani oleh plugin runtime. Lihat dokumentasi plugin untuk format payload spesifik.

## Aturan dan Batasan

| Aspek | Behavior |
|-------|----------|
| Jumlah detail per master | Tidak dibatasi validator. Plugin runtime bisa memberi batasan praktis |
| Nested detail (detail dari detail) | Tidak didukung. Hanya 1 level detail per master |
| Detail tanpa master | Tidak didukung. Detail selalu terkait master |
| `detailId` unik per page | Tidak ada validasi unique secara eksplisit di validator. Tetapi pastikan unik untuk menghindari konflik save |

## Contoh Konfigurasi Lengkap

### Sales Order dengan Order Item

```json
{
    "pageId": "sales-order",
    "pageTitle": "Sales Order",
    "pageIcon": "shopping-cart",
    "apiPath": "sales-order",
    "primaryKey": "order_id",
    "displayField": "order_no",
    "features": {
        "enableSearch": true,
        "fieldLayout": "vertical"
    },
    "fields": [
        {
            "name": "order_no",
            "label": "No Order",
            "type": "text",
            "required": true,
            "inTable": true,
            "tableOrder": 1,
            "readonly": true,
            "defaultValue": {
                "source": "idgen",
                "mode": "number",
                "resource": "sales-order",
                "format": "yyyymm",
                "numDigits": 5,
                "separator": "/",
                "reserve": true,
                "ttl": 60
            }
        },
        {
            "name": "order_date",
            "label": "Tanggal",
            "type": "date",
            "required": true,
            "inTable": true,
            "tableOrder": 2
        },
        {
            "name": "customer_id",
            "label": "Customer",
            "type": "select",
            "required": true,
            "inTable": true,
            "tableOrder": 3,
            "tableField": "customer_name",
            "dataSource": {
                "type": "api",
                "resource": "customer",
                "select": ["customer_id", "customer_name"]
            }
        }
    ],
    "fieldRows": [
        {"fields": ["order_no", "order_date"]},
        {"fields": ["customer_id"]}
    ],
    "details": [
        {
            "detailId": "items",
            "primaryKey": "order_item_id",
            "fields": [
                {
                    "name": "product_id",
                    "label": "Produk",
                    "type": "select",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1,
                    "tableField": "product_name",
                    "dataSource": {
                        "type": "api",
                        "resource": "product",
                        "select": ["product_id", "product_name"]
                    }
                },
                {
                    "name": "qty",
                    "label": "Qty",
                    "type": "number",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 2,
                    "min": 1
                },
                {
                    "name": "price",
                    "label": "Harga",
                    "type": "number",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 3
                }
            ]
        }
    ]
}
```

---

← [`features.md`](./features.md) | [Selanjutnya: `workflow-actions.md`](./workflow-actions.md) →
