# Contoh Lengkap

## Tabel Master Sederhana

```json
{
    "tableName": "category",
    "primaryKey": "category_id",
    "fieldName": [
        "category_id",
        "category_code",
        "category_name",
        "description",
        "is_active",
        "created_at",
        "created_by",
        "updated_at",
        "updated_by"
    ],
    "fieldValidation": [
        {
            "name": "category_id",
            "type": "uuid",
            "constraints": { "primaryKey": true, "autoGenerate": true }
        },
        {
            "name": "category_code",
            "type": "string",
            "constraints": { "required": true, "maxLength": 20, "unique": true, "uppercase": true, "trim": true }
        },
        {
            "name": "category_name",
            "type": "string",
            "constraints": { "required": true, "maxLength": 100 }
        },
        {
            "name": "is_active",
            "type": "boolean",
            "constraints": { "default": true }
        }
    ],
    "fieldNameLookup": {
        "id": "category_id",
        "text": "category_code||' - '||category_name as display_text"
    },
    "defaultScope": {
        "lookup": { "is_active": true },
        "read":   { "is_active": true }
    },
    "datatablesQuery": "file:query/category-datatables.sql",
    "datatablesWhere": ["category_code", "category_name", "all"],
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

## Tabel Master dengan View Query dan Export

```json
{
    "tableName": "item_product",
    "primaryKey": "item_product_id",
    "fieldName": [
        "item_product_id", "product_code", "sku", "product_name",
        "category_id", "category_name",
        "purchase_price", "selling_price", "stock", "is_active"
    ],
    "viewQuery": "file:query/item-product-detail.sql",
    "datatablesQuery": "file:query/item-product-datatables.sql",
    "datatablesWhere": ["product_code", "sku", "product_name", "category_name", "all"],
    "exportQuery": "file:query/item-product-export.sql",
    "columnFormats": {
        "purchase_price": { "type": "number", "format": "#,##0.00" },
        "selling_price":  { "type": "number", "format": "#,##0.00" },
        "stock":          { "type": "number", "format": "#,##0" }
    },
    "fieldNameLookup": {
        "id": "item_product_id",
        "text": "product_code||' - '||product_name as display_text"
    },
    "defaultScope": {
        "lookup": { "is_active": true }
    },
    "fieldValidation": [
        { "name": "item_product_id", "type": "uuid",    "constraints": { "primaryKey": true, "autoGenerate": true } },
        { "name": "product_code",    "type": "string",  "constraints": { "required": true, "maxLength": 20, "unique": true } },
        { "name": "sku",             "type": "string",  "constraints": { "required": true, "maxLength": 20, "unique": true } },
        { "name": "product_name",    "type": "string",  "constraints": { "required": true, "maxLength": 100 } },
        { "name": "category_id",     "type": "uuid",    "constraints": { "required": true } },
        { "name": "purchase_price",  "type": "decimal", "constraints": { "min": 0, "precision": 2 } },
        { "name": "selling_price",   "type": "decimal", "constraints": { "min": 0, "precision": 2 } },
        { "name": "stock",           "type": "decimal", "constraints": { "min": 0 } },
        { "name": "is_active",       "type": "boolean", "constraints": { "default": true } }
    ],
    "action": {
        "datatables": true,
        "create": true,
        "update": true,
        "delete": true,
        "first": true,
        "lookup": true,
        "read": true,
        "export": true,
        "adjust": true
    },
    "adjustConfig": {
        "fields": {
            "stock": { "type": "number", "min": 0, "allowNegativeResult": false }
        },
        "reasonRequired": true
    }
}
```

## Tabel Transaksi dengan Master Detail dan Workflow

```json
{
    "tableName": "stock_inbound",
    "primaryKey": "stock_inbound_id",
    "fieldName": [
        "stock_inbound_id", "inbound_number", "inbound_date",
        "warehouse_id", "supplier_id", "reference_number", "notes",
        "total_items", "total_qty", "total_amount", "status",
        "created_at", "created_by", "updated_at", "updated_by"
    ],
    "viewQuery": "file:query/stock-inbound-detail.sql",
    "datatablesQuery": "file:query/stock-inbound-datatables.sql",
    "datatablesWhere": ["inbound_number", "reference_number", "status", "all"],
    "dateTimeFields": {
        "inbound_date": { "type": "date", "format": "dd/MM/yyyy" }
    },
    "fieldValidation": [
        { "name": "stock_inbound_id", "type": "uuid",    "constraints": { "primaryKey": true, "autoGenerate": true } },
        { "name": "inbound_number",   "type": "string",  "constraints": { "required": true, "unique": true, "maxLength": 50 } },
        { "name": "inbound_date",     "type": "date",    "constraints": { "required": true, "format": "dd/MM/yyyy" } },
        { "name": "warehouse_id",     "type": "uuid",    "constraints": { "required": true } },
        { "name": "supplier_id",      "type": "uuid",    "constraints": { "required": true } },
        { "name": "status",           "type": "string",  "constraints": { "required": true, "enum": ["draft", "confirmed", "closed", "cancelled"], "default": "draft" } }
    ],
    "action": {
        "datatables": true,
        "create": true,
        "update": true,
        "delete": true,
        "first": true,
        "read": true,
        "createComposite": true,
        "updateComposite": true,
        "readComposite": true,
        "workflow": true
    },
    "masterDetail": {
        "enabled": true,
        "detailTable": "stock_inbound_item",
        "foreignKey": "stock_inbound_id",
        "cascadeDelete": true,
        "transactionMode": "required",
        "detailConfig": {
            "tableName": "stock_inbound_item",
            "primaryKey": "stock_inbound_item_id",
            "fieldName": [
                "stock_inbound_item_id", "stock_inbound_id", "line_number",
                "item_product_id", "qty_received", "uom",
                "unit_price", "total_amount", "notes"
            ],
            "detailQuery": "file:query/stock-inbound-item-detail.sql",
            "requiredFields": ["line_number", "item_product_id", "qty_received", "unit_price"],
            "autoCalculateFields": {
                "total_amount": { "type": "generated", "formula": "qty_received * unit_price" }
            }
        },
        "headerCalculations": {
            "total_items":  { "type": "count", "source": "items.length" },
            "total_qty":    { "type": "sum",   "source": "items.qty_received" },
            "total_amount": { "type": "sum",   "source": "items.total_amount" }
        }
    },
    "workflow": {
        "statusField": "status",
        "transitions": {
            "draft":     ["confirmed", "cancelled"],
            "confirmed": ["closed", "cancelled"],
            "closed":    [],
            "cancelled": []
        },
        "hooks": {
            "confirmed": {
                "onAfter": [
                    {
                        "type": "api",
                        "method": "POST",
                        "url": "/stock-management/process-inbound",
                        "body": {
                            "stock_inbound_id": "{{id}}",
                            "inbound_number":   "{{record.inbound_number}}",
                            "status":           "{{newStatus}}",
                            "previous_status":  "{{oldStatus}}",
                            "action":           "confirm-inbound"
                        },
                        "blocking": true,
                        "timeout":  10000
                    }
                ]
            }
        }
    }
}
```

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
