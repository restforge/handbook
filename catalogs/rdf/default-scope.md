# Field Default Scope (`defaultScope`)

Berfungsi sebagai filter otomatis yang ditambahkan ke WHERE clause pada action lookup dan read. Hanya kedua action ini yang didukung, sedangkan action baca lainnya seperti datatables dan first **tidak terpengaruh**.

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
| `defaultScope.lookup` ditetapkan | Endpoint `GET /lookup` dan `POST /lookup` terfilter secara otomatis |
| `defaultScope.read` ditetapkan | Endpoint `POST /read` terfilter secara otomatis |
| Key selain `lookup`/`read` | Warning saat generate, key diabaikan |

### Generasi Otomatis untuk `is_active`

Untuk kolom `is_active`, `defaultScope` tidak harus ditulis secara manual. Bila tabel memiliki kolom `is_active`, `payload generate` secara otomatis menyisipkan `defaultScope: { lookup: { is_active: true }, read: { is_active: true } }`, dan `payload sync` menjaganya tetap selaras (menambah saat kolom ada, melepas saat kolom dihapus). Penulisan manual tetap berlaku untuk kolom atau nilai filter lain (mis. `tenant_id`, `show_in_store`), dan sinkronisasi otomatis hanya menyentuh key `is_active` sehingga filter kustom tidak hilang. Detail: [`commands/payload/generate.md`](../../commands/restforge-backend/payload/generate.md#default-scope-otomatis-is_active) dan [`commands/payload/sync.md`](../../commands/restforge-backend/payload/sync.md#default-scope-is_active-built-in).

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
| Setiap kolom harus ada di `fieldName` | `defaultScope.<action> references column '<col>' which is not in fieldName of <file>` |
| Nilai harus boolean/string/number | `defaultScope.<action>.<col> in <file> must be a boolean, string, or number` |

Detail lengkap (interaksi dengan WHERE user, generasi SQL per dialect, query builder `.active()` di processor) dibahas di dokumentasi fitur scope terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
