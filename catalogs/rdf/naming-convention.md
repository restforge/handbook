# Konvensi Penamaan

| Element | Konvensi | Contoh |
|---------|----------|--------|
| Nama file RDF | kebab-case | `item-product.json`, `stock-inbound.json` |
| Nama endpoint URL | kebab-case (otomatis dari nama file) | `/item-product/datatables`, `/stock-inbound/create` |
| `tableName` | snake_case, singular | `category`, `stock_inbound_item` |
| `primaryKey` | snake_case, pattern `<table>_id` | `category_id`, `stock_inbound_id` |
| `fieldName` item | snake_case | `category_id`, `created_at` |
| Nama file SQL eksternal | kebab-case, sufiks deskriptif | `supplier-datatables.sql`, `item-product-detail.sql` |
| Nama folder SQL | `payload/query/` | - |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
