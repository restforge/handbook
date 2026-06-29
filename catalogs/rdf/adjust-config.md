# Field Adjust (`adjustConfig`)

Konfigurasi untuk endpoint `/adjust`, yaitu operasi atomik increment/decrement pada field numerik.

```json
{
    "action": { "adjust": true },
    "adjustConfig": {
        "fields": {
            "stock": {
                "type": "number",
                "min": 0,
                "allowNegativeResult": false
            }
        },
        "reasonRequired": true
    }
}
```

## Properti Top-Level

| Property | Tipe | Default | Keterangan |
|----------|------|---------|-----------|
| `fields` | object | `{}` | Daftar field yang boleh di-adjust |
| `reasonRequired` | boolean | `false` | Jika `true`, field `reason` di request body wajib diisi |

## Properti `fields[name]`

| Property | Tipe | Default | Keterangan |
|----------|------|---------|-----------|
| `type` | string | `"number"` | Harus `"number"`. Field non-numerik akan ditolak saat runtime |
| `min` | number | `0` | Batas minimum nilai field setelah adjustment |
| `allowNegativeResult` | boolean | `true` | Jika `false`, operasi gagal (HTTP 409) saat hasil < `min` |

Detail lengkap (lifecycle hook `onBeforeAdjust`/`onAfterAdjust`, error format, integrasi component engine) dibahas di dokumentasi fitur adjust terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
