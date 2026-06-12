# Workflow Actions — `workflow` + `workflowActions[]`

> Tombol aksi yang mengubah status record (state machine sederhana) di luar tombol standar Save/Delete.

## Konsep

Banyak record memiliki lifecycle status (Draft → Submitted → Approved → Closed). UDF mendeklarasikan tombol-tombol transisi status sebagai array `workflowActions[]`, dipasangkan dengan blok `workflow` yang menunjuk field penyimpan status.

```json
{
    "workflow": {
        "statusField": "order_status"
    },
    "workflowActions": [
        {
            "actionId": "submit",
            "label": "Submit",
            "api": {
                "endpoint": "/submit"
            }
        },
        {
            "actionId": "approve",
            "label": "Approve",
            "api": {
                "endpoint": "/approve"
            }
        }
    ]
}
```

## Blok `workflow`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `statusField` | string | ✓ jika `workflowActions[]` ada | Nama field yang menyimpan status record |

Jika `workflowActions[]` didefinisikan tetapi `workflow.statusField` tidak ada, validator menghasilkan error:

```
[<pageId>] workflow.statusField must be provided when workflowActions is defined
```

## Properti `workflowActions[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `actionId` | string | ✓ | Identifier aksi (unik per page) |
| `label` | string | ✓ | Label tombol di UI |
| `api` | object | ✓ | Konfigurasi endpoint backend untuk eksekusi aksi |
| `api.endpoint` | string | ✓ | Path endpoint untuk eksekusi aksi |
| `api.payload` | object | ✗ | Payload tambahan yang dikirim ke endpoint |
| `confirm` | object | ✗ | Konfigurasi dialog konfirmasi sebelum eksekusi |
| `confirm.summary` | array | ✗ | Daftar field yang ditampilkan di dialog konfirmasi |

### `confirm.summary[]`

| Properti | Tipe | Keterangan |
|----------|------|-----------|
| `field` | string | Nama field di `fields[]` yang nilainya ditampilkan di summary |

Jika `confirm.summary[].field` tidak ada di `fields[]`, validator menghasilkan warning:

```
[<pageId>].workflowActions[<i>] confirm.summary references field '<name>' which is not found in 'fields'
```

## Aturan Validasi

| Aturan | Error |
|--------|-------|
| `actionId` wajib non-empty | `"[<pageId>].workflowActions[<i>] actionId must be provided"` |
| `actionId` unik per page | `"[<pageId>].workflowActions[<i>] actionId '<value>' is duplicated"` |
| `label` wajib | `"[<pageId>].workflowActions[<i>] label must be provided"` |
| `api` wajib | `"[<pageId>].workflowActions[<i>] api must be provided"` |
| `api.endpoint` wajib | `"[<pageId>].workflowActions[<i>] api.endpoint must be provided"` |

## Contoh Workflow Sederhana

### Order dengan 3 Aksi

```json
{
    "pageId": "sales-order",
    "pageTitle": "Sales Order",
    "primaryKey": "order_id",
    "displayField": "order_no",
    "apiPath": "sales-order",
    "fields": [
        {"name": "order_no", "label": "No Order", "type": "text", "required": true},
        {"name": "customer_name", "label": "Customer", "type": "text"},
        {"name": "total_amount", "label": "Total", "type": "number"},
        {"name": "order_status", "label": "Status", "type": "text", "readonly": true}
    ],
    "workflow": {
        "statusField": "order_status"
    },
    "workflowActions": [
        {
            "actionId": "submit",
            "label": "Submit for Approval",
            "api": {
                "endpoint": "/submit"
            }
        },
        {
            "actionId": "approve",
            "label": "Approve",
            "api": {
                "endpoint": "/approve"
            },
            "confirm": {
                "summary": [
                    {"field": "order_no"},
                    {"field": "customer_name"},
                    {"field": "total_amount"}
                ]
            }
        },
        {
            "actionId": "reject",
            "label": "Reject",
            "api": {
                "endpoint": "/reject",
                "payload": {
                    "reason": "Insufficient stock"
                }
            }
        }
    ]
}
```

### Dialog Konfirmasi

Atribut `confirm.summary` memunculkan dialog konfirmasi sebelum aksi dieksekusi. Dialog menampilkan label dan nilai field yang disebut di summary:

```
┌────────────────────────────────────────┐
│ Confirm: Approve                       │
├────────────────────────────────────────┤
│ Apakah aksi ini akan dieksekusi?       │
│                                        │
│ No Order   : SO-2025/00001             │
│ Customer   : PT ABC Sejahtera          │
│ Total      : 1,500,000                 │
│                                        │
│ [Cancel]  [Approve]                    │
└────────────────────────────────────────┘
```

Jika `confirm` tidak ada, aksi dieksekusi langsung tanpa dialog.

## Interaksi dengan `fieldStates`

Workflow biasanya dikombinasikan dengan `fieldStates` untuk lock field tertentu setelah transisi status. Contoh: field tidak boleh diedit lagi setelah status Approved.

```json
{
    "fieldStates": [
        {
            "when": "order_status == 'approved'",
            "state": "readonly"
        }
    ]
}
```

Lihat [`field-states.md`](./field-states.md) untuk detail.

## Endpoint API yang Dipakai

Generator menggunakan pola endpoint berikut untuk eksekusi aksi:

```
POST <apiBaseUrl>/<apiPath>/<recordId><api.endpoint>
Content-Type: application/json
Body: <api.payload>  (jika ada)
```

Contoh:

```
POST http://localhost:3031/api/dbsales/sales-order/so-001/approve
Content-Type: application/json
Body: {}
```

Backend bertanggung jawab untuk:

1. Memvalidasi transisi status (mis. dari `submitted` ke `approved` valid)
2. Update field `statusField` di record
3. Menjalankan side effect (notification, audit log, downstream sync)

## Tombol Standar vs Workflow Actions

| Aspek | Tombol Standar | Workflow Actions |
|-------|----------------|------------------|
| Tombol yang dihasilkan | Save, Cancel, Delete | Tombol custom per aksi (`actionId`) |
| Endpoint | `POST /<apiPath>`, `PUT /<apiPath>/<id>`, `DELETE /<apiPath>/<id>` | `POST /<apiPath>/<id><api.endpoint>` |
| Konfirmasi | Default: konfirmasi delete | Opsional per aksi via `confirm.summary` |
| Modifikasi data | Save mengirim semua field | Workflow action hanya mengubah status |

---

← [`master-detail.md`](./master-detail.md) | [Selanjutnya: `field-states.md`](./field-states.md) →
