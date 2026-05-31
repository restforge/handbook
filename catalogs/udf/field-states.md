# Field States — `fieldStates[]`

> Aturan kondisional untuk mengubah state field (readonly/hidden) berdasarkan ekspresi data.

## Konsep

Beberapa skenario membutuhkan field tertentu menjadi readonly atau hidden pada kondisi data tertentu. Contoh:

| Skenario | Behavior yang Diinginkan |
|----------|--------------------------|
| Order sudah Approved | Semua field jadi readonly |
| Status = Draft | Field `approval_date` di-hidden |
| Tipe transaksi = Pembelian Kredit | Field `due_date` tampil; tipe Tunai → di-hidden |

UDF mendeklarasikan aturan ini di array `fieldStates[]` di page object.

## Sintaks

```json
{
    "fieldStates": [
        {
            "when": "order_status == 'approved'",
            "state": "readonly"
        },
        {
            "when": "payment_type == 'cash'",
            "state": "hidden",
            "fields": ["due_date"]
        }
    ]
}
```

## Properti `fieldStates[]`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `when` | string | ✓ | Ekspresi kondisi (JavaScript-like). Dievaluasi di runtime pada data record |
| `state` | string | ✓ | State yang diaplikasikan. Valid: `"readonly"`, `"hidden"` |
| `fields` | array of string | ✗ | Daftar field yang terpengaruh. Jika tidak ada, aturan berlaku untuk semua field |

### Valid `state`

```
readonly, hidden
```

| State | Behavior |
|-------|----------|
| `"readonly"` | Field tetap terlihat tetapi tidak bisa diedit |
| `"hidden"` | Field disembunyikan dari form |

Nilai lain menghasilkan error:

```
[<pageId>].fieldStates[<i>] state is invalid: '<value>'. Use 'readonly' or 'hidden'
```

## Sintaks Ekspresi `when`

Ekspresi `when` dievaluasi runtime di JavaScript yang di-generate. Bentuk yang umum dipakai:

| Pola | Contoh |
|------|--------|
| Kesetaraan nilai | `order_status == 'approved'` |
| Tidak sama | `payment_type != 'cash'` |
| Boolean | `is_locked == true` |
| Kombinasi AND/OR | `status == 'approved' && total > 1000000` |
| Negasi | `!is_active` |

Field yang direferensikan di `when` harus ada di `fields[]` agar nilainya tersedia saat evaluasi.

## Aturan Validasi

| Aturan | Error |
|--------|-------|
| `when` wajib non-empty | `"[<pageId>].fieldStates[<i>] when must be provided"` |
| `state` valid (`readonly` atau `hidden`) | `"[<pageId>].fieldStates[<i>] state is invalid: '<value>'. Use 'readonly' or 'hidden'"` |

Validator tidak memvalidasi sintaks ekspresi `when` (parsing kompleks ditangani runtime). Pastikan ekspresi valid JavaScript untuk menghindari runtime error.

## Scope: Per-Field vs Global

### Scope Global (Semua Field)

Tanpa atribut `fields`, aturan berlaku untuk semua field di page:

```json
{
    "fieldStates": [
        {
            "when": "order_status == 'closed'",
            "state": "readonly"
        }
    ]
}
```

Berlaku: semua field jadi readonly ketika `order_status == 'closed'`.

### Scope Per-Field

Dengan atribut `fields`, aturan hanya berlaku untuk field yang disebutkan:

```json
{
    "fieldStates": [
        {
            "when": "payment_type == 'cash'",
            "state": "hidden",
            "fields": ["due_date", "installment_count"]
        }
    ]
}
```

Berlaku: hanya `due_date` dan `installment_count` yang di-hidden ketika `payment_type == 'cash'`.

## Multiple Rules

Beberapa aturan dapat diaplikasikan bersamaan. Aturan dievaluasi berurutan, dan state yang aplikatif diakumulasi:

```json
{
    "fieldStates": [
        {
            "when": "order_status == 'draft'",
            "state": "hidden",
            "fields": ["approval_date", "approved_by"]
        },
        {
            "when": "order_status == 'approved'",
            "state": "readonly"
        }
    ]
}
```

Behavior:

| `order_status` | Hasil |
|----------------|-------|
| `'draft'` | `approval_date` dan `approved_by` di-hidden. Field lain editable |
| `'submitted'` | Semua field editable |
| `'approved'` | Semua field readonly |
| `'rejected'` | Semua field editable |

## Kombinasi dengan Workflow

`fieldStates` sangat sering dipakai bersama `workflow` + `workflowActions[]` untuk lock field setelah transisi status:

```json
{
    "workflow": {
        "statusField": "order_status"
    },
    "workflowActions": [
        {
            "actionId": "approve",
            "label": "Approve",
            "api": {"endpoint": "/approve"}
        }
    ],
    "fieldStates": [
        {
            "when": "order_status == 'approved'",
            "state": "readonly"
        }
    ]
}
```

Behavior: setelah aksi Approve berhasil, semua field jadi readonly otomatis.

## Best Practice

| Skenario | Rekomendasi |
|----------|-------------|
| Lock setelah Approved | Global `readonly` rule berdasarkan status field |
| Field optional per kondisi (mis. due_date) | Per-field `hidden` rule dengan `fields: [...]` |
| Field readonly satu arah (mis. nomor order) | Pakai `readonly: true` di field, bukan `fieldStates` |
| Mode Add vs Edit | Tidak ada konsep langsung. Pakai ekspresi `primaryKey == null` atau sejenis |

## Contoh Penggunaan Praktis

### Order Lifecycle

```json
{
    "pageId": "sales-order",
    "fields": [
        {"name": "order_no", "label": "No", "type": "text", "readonly": true},
        {"name": "customer_id", "label": "Customer", "type": "select", "dataSource": { /* ... */ }},
        {"name": "order_total", "label": "Total", "type": "number"},
        {"name": "order_status", "label": "Status", "type": "text", "readonly": true},
        {"name": "approval_date", "label": "Tgl Approve", "type": "timestamp", "readonly": true},
        {"name": "approved_by", "label": "Disetujui Oleh", "type": "text", "readonly": true}
    ],
    "workflow": {"statusField": "order_status"},
    "workflowActions": [
        {"actionId": "submit", "label": "Submit", "api": {"endpoint": "/submit"}},
        {"actionId": "approve", "label": "Approve", "api": {"endpoint": "/approve"}}
    ],
    "fieldStates": [
        {
            "when": "order_status == 'draft' || order_status == 'submitted'",
            "state": "hidden",
            "fields": ["approval_date", "approved_by"]
        },
        {
            "when": "order_status == 'approved' || order_status == 'rejected'",
            "state": "readonly"
        }
    ]
}
```

Behavior:

| `order_status` | Field `customer_id` | Field `approval_date` | Tombol |
|----------------|---------------------|----------------------|--------|
| `'draft'` | Editable | Hidden | Submit |
| `'submitted'` | Editable | Hidden | Approve |
| `'approved'` | Readonly | Readonly + Visible | (tidak ada) |
| `'rejected'` | Readonly | Readonly + Visible | (tidak ada) |

---

← [`workflow-actions.md`](./workflow-actions.md) | [Selanjutnya: `navigation.md`](./navigation.md) →
