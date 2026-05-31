# Field Default Scope (`defaultScope`)

Filter otomatis yang ditambahkan ke WHERE clause untuk action `lookup` dan `read`. Action `datatables` dan `first` **tidak terpengaruh**.

```json
{
    "defaultScope": {
        "lookup": { "is_active": true },
        "read": { "is_active": true }
    }
}
```

## Aturan Konfigurasi

| Kondisi | Perilaku |
|---------|----------|
| `defaultScope` tidak ada | Semua action tanpa filter tambahan (backward compatible) |
| `defaultScope.lookup` di-set | Endpoint `GET /lookup` dan `POST /lookup` ter-filter otomatis |
| `defaultScope.read` di-set | Endpoint `POST /read` ter-filter otomatis |
| Key selain `lookup`/`read` | Warning saat generate, key diabaikan |

## Tipe Nilai yang Didukung

| Tipe | Contoh | Catatan |
|------|--------|---------|
| `boolean` | `true`, `false` | PostgreSQL native; MySQL/Oracle dikonversi ke string `'true'`/`'false'` |
| `string` | `"active"` | Literal SQL string |
| `number` | `1`, `0` | Numeric literal |

## Validasi

| Aturan | Pesan Error |
|--------|-------------|
| `defaultScope` harus object | `defaultScope must be an object` |
| Setiap kolom harus exist di `fieldName` | `defaultScope.<action> references column '<col>' which is not in fieldName of <file>` |
| Nilai harus boolean/string/number | `defaultScope.<action>.<col> in <file> must be a boolean, string, or number` |

Detail lengkap (interaksi dengan WHERE user, generasi SQL per dialect, query builder `.active()` di processor) dibahas di dokumentasi fitur scope terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
