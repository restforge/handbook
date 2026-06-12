# Sidebar Navigation — `navigation`

> Konfigurasi manual sidebar navigation. Override otomatis auto-derive dari `pageGroup`.

## Konsep

UDF mendukung dua mode pembentukan sidebar navigation:

| Mode | Trigger | Sumber |
|------|---------|--------|
| Auto-derive | Tidak ada blok `navigation` di root | `pageGroup` di setiap page |
| Manual | Ada blok `navigation` di root | Struktur eksplisit di `navigation.items[]` |

Jika kedua mekanisme ada (blok `navigation` ada DAN ada `pageGroup` di page), validator menghasilkan warning bahwa `pageGroup` akan diabaikan.

## Sintaks

```json
{
    "navigation": {
        "items": [
            {
                "type": "page",
                "pageRef": "dashboard",
                "icon": "home",
                "label": "Dashboard"
            },
            {
                "type": "group",
                "label": "Master Data",
                "icon": "database",
                "children": [
                    {"type": "page", "pageRef": "customer", "label": "Customer"},
                    {"type": "page", "pageRef": "supplier", "label": "Supplier"},
                    {"type": "page", "pageRef": "product", "label": "Product"}
                ]
            },
            {"type": "separator"},
            {
                "type": "group",
                "label": "Transaksi",
                "icon": "file-text",
                "children": [
                    {
                        "type": "page",
                        "pageRef": "sales-order",
                        "label": "Sales Order",
                        "badgeColor": "warning"
                    }
                ]
            }
        ]
    }
}
```

## Properti `navigation`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `items` | array | ✓ | Array item navigation (minimal 1 entry) |

## Properti `navigation.items[]`

Tiga tipe item didukung: `page`, `group`, `separator`.

### Tipe `page`

Link langsung ke halaman.

```json
{
    "type": "page",
    "pageRef": "customer",
    "label": "Customer",
    "icon": "users",
    "badgeColor": "info"
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `type` | string | ✓ | Harus `"page"` |
| `pageRef` | string | ✓ | `pageId` yang dirujuk. Harus ada di `pages[]` |
| `label` | string | ✗ | Override label. Default: `pageTitle` dari page yang dirujuk |
| `icon` | string | ✗ | Override ikon. Hanya berlaku di depth 1 (top-level) |
| `badgeColor` | string | ✗ | Warna badge. Valid: `danger`, `info`, `primary`, `success`, `warning` |

### Tipe `group`

Folder yang berisi sub-item.

```json
{
    "type": "group",
    "label": "Master Data",
    "icon": "database",
    "children": [
        {"type": "page", "pageRef": "customer"},
        {"type": "page", "pageRef": "supplier"}
    ]
}
```

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `type` | string | ✓ | Harus `"group"` |
| `label` | string | ✓ | Label folder di sidebar |
| `icon` | string | ✗ | Ikon folder. Hanya berlaku di depth 1 |
| `children` | array | ✓ | Array item child (minimal 1 entry, non-empty) |

### Tipe `separator`

Garis pemisah visual.

```json
{"type": "separator"}
```

Tidak memerlukan properti tambahan.

## Valid Type dan Color

| Field | Nilai Valid |
|-------|-------------|
| `type` | `group`, `page`, `separator` |
| `badgeColor` | `danger`, `info`, `primary`, `success`, `warning` |

Nilai di luar daftar valid menghasilkan error spesifik.

## Maksimal Kedalaman

Sidebar mendukung maksimal 3 level kedalaman (`NAV_MAX_DEPTH = 3`):

```
Level 1: Item top-level (dengan icon)
  Level 2: Item child dari group level 1
    Level 3: Item child dari group level 2
```

Lebih dari 3 level menghasilkan error:

```
<path> exceeds the maximum nesting depth of 3. Flatten the structure or remove the deepest group.
```

## Aturan Icon

Atribut `icon` hanya dirender di depth 1 (top-level). Icon di depth lebih dalam menghasilkan warning:

```
<path>.icon is set at depth <n> but icons are only rendered at depth 1; this icon will be ignored.
```

## Aturan Validasi

| Aturan | Hasil |
|--------|-------|
| `navigation` harus object | Error: `"navigation must be an object, got <type>"` |
| `navigation.items` harus array | Error: `"navigation.items must be an array of navigation items"` |
| `type` valid | Error: `"<path>.type is invalid: '<value>'. Valid types: group, page, separator"` |
| `badgeColor` valid jika ada | Error: `"<path>.badgeColor is invalid: '<value>'. Valid colors: danger, info, primary, success, warning"` |
| `pageRef` wajib untuk type `page` | Error: `"<path>.pageRef must be provided when type='page'"` |
| `pageRef` harus ada di `pages[]` (scope `app`) | Error: `"<path>.pageRef '<value>' does not match any pageId in 'pages'. Add the page or fix the reference."` |
| `pageRef` orphan di scope `form` | Warning: `"<path>.pageRef '<value>' does not match any pageId in 'pages'. Sidebar will filter at runtime based on existing output files."` |
| `label` wajib untuk type `group` | Error: `"<path>.label must be provided when type='group'"` |
| `children` non-empty untuk type `group` | Error: `"<path>.children must be a non-empty array when type='group'"` |
| Depth maks 3 | Error: `"<path> exceeds the maximum nesting depth of 3..."` |

## Auto-Derive dari `pageGroup`

Jika tidak ada blok `navigation` di root, sidebar dibentuk otomatis dari `pageGroup` di setiap page:

```json
{
    "pages": [
        {"pageId": "customer", "pageGroup": ["Master"], "pageTitle": "Customer"},
        {"pageId": "supplier", "pageGroup": ["Master"], "pageTitle": "Supplier"},
        {"pageId": "sales-order", "pageGroup": ["Transaksi", "Penjualan"], "pageTitle": "SO"},
        {"pageId": "purchase-order", "pageGroup": ["Transaksi", "Pembelian"], "pageTitle": "PO"}
    ]
}
```

Hasil sidebar auto-derive:

```
Master
├── Customer
└── Supplier

Transaksi
├── Penjualan
│   └── SO
└── Pembelian
    └── PO
```

Maksimal 2 level di `pageGroup` (page sendiri menempati level ke-3).

## Coexistence dengan `pageGroup`

Jika `navigation` ada DAN `pageGroup` juga didefinisikan di salah satu page, validator menghasilkan warning:

```
pageGroup is defined on the following pages but will be ignored because manual 'navigation'
block is present: <pageIds>. Remove the manual navigation block to use auto-derive,
or remove pageGroup to silence this warning.
```

Pilih salah satu mekanisme untuk menghindari kebingungan.

## Scope `app` vs `form` (Generate Mode)

Validasi `pageRef` berbeda antara scope `app` (generate semua page) dan `form` (generate satu page saja):

| Scope | Perilaku jika `pageRef` orphan |
|-------|-------------------------------|
| `app` | Error: page wajib ada di `pages[]` |
| `form` | Warning: sidebar akan difilter runtime berdasarkan file output yang ada |

Hal ini memungkinkan generate per-page tanpa harus regenerate sidebar setiap kali.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Aplikasi sederhana 1–5 page | Auto-derive cukup. Tulis `pageGroup` di setiap page |
| Aplikasi 10+ page dengan grup logis | Manual `navigation` untuk kontrol penuh urutan |
| Butuh separator antar grup | Manual `navigation` |
| Butuh badge berwarna di item tertentu | Manual `navigation` |
| Plugin auth dengan permission filtering | Manual `navigation` (plugin runtime filter berdasarkan permission) |

---

← [`field-states.md`](./field-states.md) | [Selanjutnya: `homepage.md`](./homepage.md) →
