# Field Kafka (`kafka`)

Konfigurasi Kafka event publishing pada operasi CRUD.

```json
{
    "kafka": {
        "enabled": true,
        "topic": "inventory.stock_inbound",
        "keyField": "stock_inbound_id",
        "publishOn": {
            "insert": true,
            "update": true,
            "delete": false
        }
    }
}
```

| Property | Tipe | Keterangan |
|----------|------|-----------|
| `enabled` | boolean | Aktifkan Kafka publishing |
| `topic` | string | Nama Kafka topic tujuan |
| `keyField` | string | Field di record yang dipakai sebagai message key (umumnya PK) |
| `publishOn.insert` | boolean | Publish event saat insert |
| `publishOn.update` | boolean | Publish event saat update |
| `publishOn.delete` | boolean | Publish event saat delete |

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
