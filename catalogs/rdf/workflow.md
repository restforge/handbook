# Field Workflow (`workflow`)

Konfigurasi endpoint `/change-status` untuk state machine dengan validasi transisi dan hook API call.

```json
{
    "action": { "workflow": true },
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
                "onBefore": [],
                "onAfter": [
                    {
                        "type": "api",
                        "method": "POST",
                        "url": "/stock-management/process-inbound",
                        "body": {
                            "stock_inbound_id": "{{id}}",
                            "status": "{{newStatus}}",
                            "previous_status": "{{oldStatus}}"
                        },
                        "blocking": true,
                        "timeout": 10000
                    }
                ]
            }
        }
    }
}
```

## Properti Top-Level `workflow`

| Property | Tipe | Default | Keterangan |
|----------|------|---------|-----------|
| `statusField` | string | `"status"` | Nama kolom database yang menyimpan status. Wajib exist di `fieldName` |
| `transitions` | object | `null` | Map transisi yang diizinkan. Jika `null`, transisi bebas tanpa validasi |
| `hooks` | object | `{}` | Map hook per target status |

## `transitions`

Object dengan key = status saat ini, value = array status tujuan yang diizinkan. Status tujuan yang tidak terdaftar akan ditolak (HTTP 400).

```json
{
    "draft": ["confirmed", "cancelled"],
    "closed": []
}
```

## `hooks[target_status]`

Object dengan key = **target status** (bukan nama action), value = object berisi `onBefore` dan/atau `onAfter` (masing-masing array of hook configuration).

| Timing | Eksekusi | Behavior saat blocking hook gagal |
|--------|----------|------------------------------------|
| `onBefore` | Sebelum SQL UPDATE | Transaction rollback, status tidak berubah |
| `onAfter` | Setelah SQL UPDATE, sebelum COMMIT | Transaction rollback, status dikembalikan |

## Hook Configuration per Item

| Property | Tipe | Default | Keterangan |
|----------|------|---------|-----------|
| `type` | string | - | Saat ini hanya `"api"` |
| `method` | enum | `"POST"` | `POST`, `PUT`, atau `PATCH` |
| `url` | string | - | URL endpoint. Mendukung full URL, absolute path (`/api/...`), atau short path (`/{endpoint}/{action}`) |
| `body` | object | `{}` | Request body dengan template variable (`{{id}}`, `{{newStatus}}`, `{{oldStatus}}`, `{{record.<field>}}`) |
| `headers` | object | `{}` | Custom HTTP headers |
| `blocking` | boolean | `false` | `true` = rollback jika gagal, `false` = fire-and-forget |
| `timeout` | number | `10000` | Timeout dalam milidetik |

Detail lengkap (template variable, concurrency contract, integrasi component engine `onBeforeWorkflow`/`onAfterWorkflow`) dibahas di dokumentasi fitur workflow terpisah.

---

**Lihat juga**: [`rdf/`](./) · [`catalogs/`](../) · [`README`](../../README.md)
