# Konvensi Penamaan

| Element | Konvensi | Contoh |
|---------|----------|--------|
| Nama tabel | snake_case, singular | `category`, `stock_inbound_item` |
| Nama field | snake_case | `category_id`, `created_at` |
| Primary key | `<table>_id` | `category_id`, `stock_inbound_id` |
| Foreign key | sama dengan PK target | FK ke `category.category_id` dinamai `category_id` |
| Unique constraint name | auto-generate `uq_<table>_<column>` | `uq_category_code` |
| Index name | auto-generate `idx_<table>_<column>` | `idx_category_code` |
| Foreign key constraint name | auto-generate `fk_<table>_<relation_key>` | `fk_purchase_order_creator`, `fk_stock_inbound_supplier` |

Auto-generated constraint name dibatasi panjang 30 karakter untuk kompatibilitas Oracle (versi sebelum 12c). Jika nama auto-generated melebihi limit, akan dipersingkat dengan algoritma deterministik (MD5 hash 8 karakter).

---

**Lihat juga**: [`sdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
