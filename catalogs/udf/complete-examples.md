# Contoh Lengkap UDF

> Contoh UDF utuh untuk skenario umum: single-page CRUD, multi-page dengan sidebar, master-detail, dashboard, dan workflow.

## Contoh 1: Single Page CRUD Sederhana

UDF minimal untuk aplikasi satu halaman dengan field standar.

```json
{
    "appConfig": {
        "appName": "Contact Management",
        "appCode": "contact-mgmt",
        "plugin": "vanilla-js-basic",
        "apiBaseUrl": "http://localhost:3031/api/dbcontact",
        "port": 3000
    },
    "pages": [
        {
            "pageId": "contact",
            "pageTitle": "Contact",
            "pageSubtitle": "Kelola data kontak",
            "pageIcon": "users",
            "apiPath": "contact",
            "primaryKey": "contact_id",
            "displayField": "contact_name",
            "features": {
                "enableSearch": true,
                "enableStatusFilter": true,
                "statusFilter": {
                    "field": "is_active",
                    "label": "Status",
                    "options": [
                        {"value": "true", "text": "Active"},
                        {"value": "false", "text": "Inactive"}
                    ]
                }
            },
            "fields": [
                {
                    "name": "contact_name",
                    "label": "Nama Kontak",
                    "type": "text",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1,
                    "maxlength": 255
                },
                {
                    "name": "email",
                    "label": "Email",
                    "type": "text",
                    "inTable": true,
                    "tableOrder": 2,
                    "placeholder": "email@contoh.com"
                },
                {
                    "name": "phone",
                    "label": "Telepon",
                    "type": "text",
                    "inTable": true,
                    "tableOrder": 3,
                    "placeholder": "+62xxx"
                },
                {
                    "name": "address",
                    "label": "Alamat",
                    "type": "textarea",
                    "rows": 3,
                    "maxlength": 500
                },
                {
                    "name": "is_active",
                    "label": "Status",
                    "type": "checkbox",
                    "inTable": true,
                    "tableOrder": 4,
                    "defaultValue": true,
                    "checkboxText": {
                        "checked": "Active",
                        "unchecked": "Inactive"
                    }
                }
            ],
            "fieldRows": [
                {"fields": ["contact_name", "email"]},
                {"fields": ["phone", "is_active"]},
                {"fields": ["address"]}
            ]
        }
    ]
}
```

## Contoh 2: Multi-Page dengan Auto-Derived Sidebar

UDF dengan beberapa page, sidebar dibentuk otomatis dari `pageGroup`.

```json
{
    "appConfig": {
        "appName": "Inventory System",
        "appCode": "inventory",
        "plugin": "vanilla-js-auth",
        "apiBaseUrl": "http://localhost:3031/api/inventory",
        "port": 3000
    },
    "homepage": "category",
    "pages": [
        {
            "pageId": "category",
            "pageTitle": "Kategori",
            "pageIcon": "tag",
            "pageGroup": ["Master Data"],
            "apiPath": "category",
            "primaryKey": "category_id",
            "displayField": "category_name",
            "features": {"enableSearch": true},
            "fields": [
                {
                    "name": "category_name",
                    "label": "Nama Kategori",
                    "type": "text",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1
                }
            ]
        },
        {
            "pageId": "product",
            "pageTitle": "Produk",
            "pageIcon": "package",
            "pageGroup": ["Master Data"],
            "apiPath": "product",
            "primaryKey": "product_id",
            "displayField": "product_name",
            "features": {"enableSearch": true},
            "fields": [
                {
                    "name": "product_name",
                    "label": "Nama Produk",
                    "type": "text",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1
                },
                {
                    "name": "category_id",
                    "label": "Kategori",
                    "type": "select",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 2,
                    "tableField": "category_name",
                    "dataSource": {
                        "type": "api",
                        "resource": "category",
                        "select": ["category_id", "category_name"]
                    }
                }
            ]
        },
        {
            "pageId": "supplier",
            "pageTitle": "Supplier",
            "pageIcon": "truck",
            "pageGroup": ["Master Data"],
            "apiPath": "supplier",
            "primaryKey": "supplier_id",
            "displayField": "supplier_name",
            "features": {"enableSearch": true},
            "fields": [
                {
                    "name": "supplier_name",
                    "label": "Nama Supplier",
                    "type": "text",
                    "required": true,
                    "inTable": true,
                    "tableOrder": 1
                }
            ]
        }
    ]
}
```

Sidebar yang dihasilkan:

```
Master Data
├── Kategori
├── Produk
└── Supplier
```

## Contoh 3: Manual Navigation dengan Separator dan Badge

UDF dengan blok `navigation` eksplisit untuk kontrol penuh urutan dan style.

```json
{
    "appConfig": {
        "appName": "Sales System",
        "appCode": "sales",
        "plugin": "vanilla-js-auth",
        "apiBaseUrl": "http://localhost:3031/api/sales",
        "port": 3000
    },
    "navigation": {
        "items": [
            {
                "type": "page",
                "pageRef": "dashboard",
                "icon": "home",
                "label": "Dashboard"
            },
            {"type": "separator"},
            {
                "type": "group",
                "label": "Master Data",
                "icon": "database",
                "children": [
                    {"type": "page", "pageRef": "customer", "label": "Customer"},
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
                    },
                    {
                        "type": "page",
                        "pageRef": "invoice",
                        "label": "Invoice",
                        "badgeColor": "success"
                    }
                ]
            }
        ]
    },
    "homepage": "dashboard",
    "pages": [
        { "include": "pages/dashboard.json" },
        { "include": "pages/customer.json" },
        { "include": "pages/product.json" },
        { "include": "pages/sales-order.json" },
        { "include": "pages/invoice.json" }
    ]
}
```

## Contoh 4: Master-Detail (Sales Order)

UDF dengan pola master-detail untuk sales order yang punya banyak order item.

```json
{
    "appConfig": {
        "appName": "Sales System",
        "appCode": "sales",
        "plugin": "vanilla-js-auth",
        "apiBaseUrl": "http://localhost:3031/api/sales",
        "port": 3000
    },
    "pages": [
        {
            "pageId": "sales-order",
            "pageTitle": "Sales Order",
            "pageIcon": "shopping-cart",
            "apiPath": "sales-order",
            "primaryKey": "order_id",
            "displayField": "order_no",
            "features": {
                "enableSearch": true,
                "enableDataFilter": true,
                "dataFilters": [
                    {
                        "name": "customer_id",
                        "label": "Customer",
                        "dataSource": {
                            "type": "api",
                            "resource": "customer",
                            "select": ["customer_id", "customer_name"]
                        }
                    }
                ]
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
                },
                {
                    "name": "remark",
                    "label": "Catatan",
                    "type": "textarea",
                    "rows": 3
                }
            ],
            "fieldRows": [
                {"fields": ["order_no", "order_date"]},
                {"fields": ["customer_id"]},
                {"fields": ["remark"]}
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
                        },
                        {
                            "name": "subtotal",
                            "label": "Subtotal",
                            "type": "number",
                            "inTable": true,
                            "tableOrder": 4,
                            "readonly": true
                        }
                    ]
                }
            ]
        }
    ]
}
```

## Contoh 5: Workflow dengan Field States

UDF untuk approval workflow dengan field lock setelah Approved.

```json
{
    "appConfig": {
        "appName": "Purchase System",
        "appCode": "purchase",
        "plugin": "vanilla-js-auth",
        "apiBaseUrl": "http://localhost:3031/api/purchase",
        "port": 3000
    },
    "pages": [
        {
            "pageId": "purchase-order",
            "pageTitle": "Purchase Order",
            "apiPath": "purchase-order",
            "primaryKey": "po_id",
            "displayField": "po_no",
            "features": {"enableSearch": true},
            "fields": [
                {"name": "po_no", "label": "No PO", "type": "text", "required": true, "readonly": true, "inTable": true, "tableOrder": 1},
                {"name": "supplier_id", "label": "Supplier", "type": "select", "required": true, "inTable": true, "tableOrder": 2, "tableField": "supplier_name", "dataSource": {"type": "api", "resource": "supplier"}},
                {"name": "total_amount", "label": "Total", "type": "number", "inTable": true, "tableOrder": 3},
                {"name": "po_status", "label": "Status", "type": "text", "readonly": true, "inTable": true, "tableOrder": 4},
                {"name": "approved_at", "label": "Tgl Approval", "type": "timestamp", "readonly": true},
                {"name": "approved_by", "label": "Disetujui Oleh", "type": "text", "readonly": true}
            ],
            "workflow": {
                "statusField": "po_status"
            },
            "workflowActions": [
                {
                    "actionId": "submit",
                    "label": "Submit for Approval",
                    "api": {"endpoint": "/submit"}
                },
                {
                    "actionId": "approve",
                    "label": "Approve",
                    "api": {"endpoint": "/approve"},
                    "confirm": {
                        "summary": [
                            {"field": "po_no"},
                            {"field": "supplier_id"},
                            {"field": "total_amount"}
                        ]
                    }
                },
                {
                    "actionId": "reject",
                    "label": "Reject",
                    "api": {"endpoint": "/reject"}
                }
            ],
            "fieldStates": [
                {
                    "when": "po_status == 'draft' || po_status == 'submitted'",
                    "state": "hidden",
                    "fields": ["approved_at", "approved_by"]
                },
                {
                    "when": "po_status == 'approved' || po_status == 'rejected'",
                    "state": "readonly"
                }
            ]
        }
    ]
}
```

## Contoh 6: Dashboard

UDF dengan halaman dashboard yang berisi widget chart dan KPI.

```json
{
    "appConfig": {
        "appName": "Sales Dashboard",
        "appCode": "sales",
        "plugin": "vanilla-js-basic",
        "apiBaseUrl": "http://localhost:3031/api/sales",
        "port": 3000
    },
    "homepage": "overview",
    "pages": [
        {
            "pageId": "overview",
            "pageType": "dashboard",
            "pageTitle": "Sales Overview",
            "pageSubtitle": "Performance dashboard",
            "pageIcon": "bar-chart-2",
            "dataSources": {
                "kpi": {
                    "url": "/api/sales/dashboard/kpi",
                    "method": "GET"
                },
                "revenueMonthly": {
                    "url": "/api/sales/dashboard/revenue-monthly",
                    "method": "GET"
                },
                "salesByRegion": {
                    "url": "/api/sales/dashboard/sales-by-region",
                    "method": "POST",
                    "body": {
                        "year": 2025
                    }
                }
            },
            "rows": [
                {
                    "gap": "1rem",
                    "columns": [
                        {
                            "colSpan": {"base": 12, "md": 4},
                            "widgets": [
                                {
                                    "widgetId": "kpi-revenue",
                                    "widgetType": "progress",
                                    "title": "Total Revenue YTD",
                                    "dataSource": "kpi"
                                }
                            ]
                        },
                        {
                            "colSpan": {"base": 12, "md": 4},
                            "widgets": [
                                {
                                    "widgetId": "kpi-orders",
                                    "widgetType": "progress",
                                    "title": "Total Orders YTD",
                                    "dataSource": "kpi"
                                }
                            ]
                        },
                        {
                            "colSpan": {"base": 12, "md": 4},
                            "widgets": [
                                {
                                    "widgetId": "kpi-customers",
                                    "widgetType": "progress",
                                    "title": "Active Customers",
                                    "dataSource": "kpi"
                                }
                            ]
                        }
                    ]
                },
                {
                    "gap": "1rem",
                    "columns": [
                        {
                            "colSpan": {"base": 12, "lg": 8},
                            "widgets": [
                                {
                                    "widgetId": "revenue-chart",
                                    "widgetType": "column",
                                    "title": "Revenue per Bulan",
                                    "subtitle": "Year-to-date",
                                    "dataSource": "revenueMonthly",
                                    "chartEngine": "apexcharts",
                                    "chart": {
                                        "xField": "month",
                                        "yField": "revenue",
                                        "format": {
                                            "yAxis": {
                                                "prefix": "Rp ",
                                                "decimalPlaces": 0
                                            },
                                            "tooltip": {
                                                "prefix": "Rp ",
                                                "decimalPlaces": 2
                                            }
                                        }
                                    }
                                }
                            ]
                        },
                        {
                            "colSpan": {"base": 12, "lg": 4},
                            "widgets": [
                                {
                                    "widgetId": "region-pie",
                                    "widgetType": "pie",
                                    "title": "Sales by Region",
                                    "dataSource": "salesByRegion",
                                    "chartEngine": "amcharts",
                                    "chart": {
                                        "xField": "region",
                                        "yField": "revenue",
                                        "shape": "donut"
                                    }
                                }
                            ]
                        }
                    ]
                }
            ]
        }
    ]
}
```

## Contoh 7: Aplikasi dengan `extends`

UDF dipecah menjadi file base + file page individual.

**File `app-config.json` (base):**

```json
{
    "appConfig": {
        "appName": "Warehouse Management",
        "appCode": "wms",
        "plugin": "vanilla-js-auth",
        "apiBaseUrl": "http://localhost:3031/api/wms",
        "port": 3000
    }
}
```

**File `wms.json` (utama):**

```json
{
    "extends": "app-config.json",
    "homepage": "stock-overview",
    "pages": [
        { "include": "pages/stock-overview.json" },
        { "include": "pages/stock-inbound.json" },
        { "include": "pages/stock-outbound.json" },
        { "include": "pages/master-product.json" },
        { "include": "pages/master-warehouse.json" }
    ]
}
```

**File `pages/master-product.json`:**

```json
{
    "pageId": "master-product",
    "pageTitle": "Master Produk",
    "pageGroup": ["Master Data"],
    "apiPath": "master-product",
    "primaryKey": "product_id",
    "displayField": "product_name",
    "features": {"enableSearch": true},
    "fields": [
        {
            "name": "product_code",
            "label": "Kode Produk",
            "type": "text",
            "required": true,
            "inTable": true,
            "tableOrder": 1,
            "maxlength": 20
        },
        {
            "name": "product_name",
            "label": "Nama Produk",
            "type": "text",
            "required": true,
            "inTable": true,
            "tableOrder": 2,
            "maxlength": 100
        },
        {
            "name": "unit",
            "label": "Satuan",
            "type": "select",
            "required": true,
            "inTable": true,
            "tableOrder": 3,
            "dataSource": {
                "type": "static",
                "options": [
                    {"value": "pcs", "text": "PCS"},
                    {"value": "box", "text": "Box"},
                    {"value": "kg", "text": "Kg"}
                ]
            }
        }
    ]
}
```

## Tip Penulisan UDF

| Skenario | Tip |
|----------|-----|
| UDF baru dari nol | Mulai dari Contoh 1, tambahkan fitur bertahap |
| Migrasi dari RDF | Pakai `npx restforge payload migrate` (binary backend) untuk konversi awal, edit hasil sesuai kebutuhan UI |
| Multi-page besar | Pakai `extends` + `include` untuk pemecahan file per modul |
| Tim besar | Pakai blok `navigation` manual untuk konsistensi urutan menu |
| Aplikasi production | Selalu pakai `wss://` untuk Live Sync, plugin `vanilla-js-auth` untuk autentikasi |

---

← [`validation-rules.md`](./validation-rules.md) | [README](./README.md) →
