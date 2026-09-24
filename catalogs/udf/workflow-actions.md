# Workflow Actions — `workflow` + `workflowActions[]`

> Tombol transisi status record (state machine sederhana) yang muncul di dialog **Change Status**, terpisah dari tombol simpan dan hapus bawaan form.

## Konsep

Record yang punya lifecycle status (Pending → Paid → Shipped → Completed) membutuhkan aksi yang mengubah status tanpa membuka form edit. UDF mendeklarasikannya lewat dua blok pada page:

- `workflow` menunjuk field penyimpan status dan peta transisi yang diizinkan.
- `workflowActions[]` mendefinisikan satu tombol per status tujuan: label, gaya, dialog konfirmasi, endpoint yang dipanggil, dan cara menampilkan hasilnya.

Pada aplikasi hasil generate, menu **Actions** di setiap baris tabel memuat item **Change Status**. Item ini membuka dialog yang menampilkan nama record (`displayField`) dan status saat ini, lalu merender tombol untuk setiap status tujuan yang diizinkan dari status tersebut.

```json
{
    "workflow": {
        "statusField": "status",
        "transitions": {
            "pending": ["paid", "cancelled"],
            "paid": ["shipped"],
            "shipped": [],
            "cancelled": []
        }
    },
    "workflowActions": [
        {
            "actionId": "paid",
            "label": "Paid",
            "style": "success",
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "paid" }
            }
        },
        {
            "actionId": "shipped",
            "label": "Shipped",
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "shipped" }
            }
        },
        {
            "actionId": "cancelled",
            "label": "Cancelled",
            "style": "danger",
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "cancelled" }
            }
        }
    ]
}
```

Dengan konfigurasi di atas, record berstatus `pending` menampilkan tombol **Paid** dan **Cancelled**; record berstatus `paid` hanya menampilkan **Shipped**; record `shipped` dan `cancelled` menampilkan teks `No transitions available for this status.`

## Blok `workflow`

| Properti | Tipe | Wajib | Keterangan |
|----------|------|:-----:|-----------|
| `statusField` | string | ✓ jika `workflowActions[]` ada | Nama field penyimpan status. Nilainya dibaca dari data baris tabel untuk menentukan tombol yang tampil, jadi kolom ini harus ikut dikembalikan endpoint `/datatables` |
| `transitions` | object | ✗ | Peta `status saat ini` → daftar `status tujuan`. Tombol hanya dirender untuk status tujuan yang ada di daftar ini **dan** punya aksi dengan `actionId` yang sama. Tanpa `transitions`, dialog selalu menampilkan `No transitions available for this status.` |

`transitions` di UDF hanya mengatur tombol yang tampil. Validasi transisi yang sesungguhnya tetap dilakukan backend lewat `workflow.transitions` di RDF (lihat [`../rdf/workflow.md`](../rdf/workflow.md)), dan penolakannya dibalas HTTP 422. Isi kedua peta ini harus sama agar tombol yang tampil selalu diterima backend.

Jika `workflowActions[]` didefinisikan tetapi `workflow.statusField` tidak ada, validator menghasilkan error:

```
[<pageId>] workflow.statusField must be provided when workflowActions is defined
```

## Properti `workflowActions[]`

| Properti | Tipe | Wajib | Default | Keterangan |
|----------|------|:-----:|---------|-----------|
| `actionId` | string | ✓ | — | **Harus sama dengan status tujuan** pada `workflow.transitions`, karena tombol dicari berdasarkan status tujuan. Unik per page |
| `label` | string | ✓ | — | Teks tombol |
| `icon` | string | ✗ | — | Nama ikon tanpa prefiks (mis. `check`, `cross`, `delivery`). Plugin memetakannya ke set ikon masing-masing: `vanilla-js-auth` memakai KeenIcons (`ki-<icon>`), `vanilla-js-custom` memetakan ke Bootstrap Icons |
| `style` | string | ✗ | `primary` | Warna tombol: `primary`, `success`, `danger`, `secondary`, `info`, `warning`. Warna yang sama dipakai untuk tombol konfirmasi bila `confirm` diisi |
| `confirm` | object | ✗ | — | Dialog konfirmasi sebelum request dikirim. Tanpa blok ini aksi langsung dieksekusi |
| `confirm.title` | string | ✗ | `Confirm?` | Judul dialog |
| `confirm.message` | string | ✗ | — | Teks penjelasan di bawah judul |
| `confirm.confirmButton` | string | ✗ | `Yes, Proceed` | Label tombol lanjut |
| `confirm.cancelButton` | string | ✗ | `Cancel` | Label tombol batal |
| `api` | object | ✓ | — | Endpoint yang dipanggil saat aksi dieksekusi. Minimal salah satu dari `endpoint` atau `payload` harus diisi |
| `api.endpoint` | string | ✗ | `<apiPath>/change-status` | Path relatif terhadap `appConfig.apiBaseUrl`, tanpa garis miring di depan |
| `api.payload` | object | ✗ | `{}` | Body request. Nilai string `"$primaryKey"` diganti dengan primary key record yang dipilih |
| `onSuccess.notification` | object | ✗ | — | Notifikasi saat response `success: true`; lihat [Tampilan Hasil](#tampilan-hasil) |
| `onSuccess.notification.type` | string | ✗ | `success` | `success`, `info`, `warning`, `error`, atau `secondary` |
| `onSuccess.notification.message` | string | ✗ | `message` dari response, atau `Status updated.` | Teks notifikasi |
| `onError.display` | string | ✗ | `modal` | Cara menampilkan penolakan atau kegagalan: `modal` atau `toast`; lihat [Tampilan Hasil](#tampilan-hasil) |
| `onError.title` | string | ✗ | `Error` | Judul modal atau toast error |

`confirm.summary[]` (daftar `{ "field": "<nama>" }`) diterima validator dan menghasilkan warning bila field tidak ada di `fields[]`, tetapi generator saat ini belum menampilkannya di dialog. Untuk menyebut nilai record pada dialog, tulis langsung di `confirm.message`.

## Alur Eksekusi

1. Pengguna membuka menu **Actions** pada baris tabel dan memilih **Change Status**.
2. Dialog **Change Status** menampilkan `displayField` record, status saat ini, dan tombol untuk setiap status tujuan pada `workflow.transitions[<status saat ini>]` yang punya aksi ber-`actionId` sama.
3. Bila aksi punya `confirm`, dialog konfirmasi tampil di atas dialog Change Status; membatalkannya mengembalikan pengguna ke dialog Change Status. Setelah dikonfirmasi, atau langsung bila tanpa `confirm`, dialog Change Status ditutup.
4. Aplikasi mengirim `POST <apiBaseUrl>/<api.endpoint>` dengan body `api.payload` yang sudah disubstitusi. Header auth ikut dikirim bila plugin memakai auth.
5. Response `success: true`: tabel dimuat ulang tanpa mengubah halaman aktif, lalu notifikasi `onSuccess.notification` tampil.
6. Response `success: false` atau status HTTP error (400, 422, 502, dan lainnya): `message` dari response ditampilkan sesuai `onError.display`. Bila response tidak punya `message`, teks `Action failed.` yang dipakai.

## Tampilan Hasil

### Sukses

Notifikasi toast di bawah tengah layar, hilang sendiri setelah beberapa detik. Judulnya selalu `Success`; tipe dan teksnya mengikuti `onSuccess.notification`.

### Gagal

Penolakan aksi workflow hampir selalu berasal dari business rule di backend (transisi tidak valid, stok tidak mencukupi, dokumen belum lengkap), yaitu pesan yang harus dibaca dan ditindaklanjuti pengguna. Karena itu default-nya modal yang menetap sampai ditutup, sama seperti error hapus di halaman yang sama.

| `onError.display` | Tampilan | Kapan cocok |
|-------------------|----------|-------------|
| `modal` (default) | Dialog SweetAlert berjudul `onError.title`, ikon error, harus ditutup pengguna | Pesan panjang atau yang menuntut tindakan |
| `toast` | Toast error di bawah tengah layar dengan tombol tutup dan **tidak hilang sendiri** | Pesan singkat yang tidak boleh menutupi tabel |

```json
{
    "actionId": "paid",
    "label": "Paid",
    "api": { "endpoint": "sales-order/change-status", "payload": { "sales_order_id": "$primaryKey", "status": "paid" } },
    "onError": { "display": "toast", "title": "Payment rejected" }
}
```

Pesan yang ditampilkan adalah `message` dari response apa adanya. Untuk penolakan hook backend, `message` berisi pesan asli hook tanpa nama fungsi atau path file handler (lihat [`../../api-spec/endpoint-change-status.md`](../../api-spec/endpoint-change-status.md)).

## Endpoint API yang Dipakai

```
POST <apiBaseUrl>/<api.endpoint>
Content-Type: application/json
Body: <api.payload> dengan "$primaryKey" diganti primary key record
```

Contoh dengan `apiBaseUrl` `http://localhost:3031/api/ecommerce`, `api.endpoint` `sales-order/change-status`, dan record `SO-001`:

```
POST http://localhost:3031/api/ecommerce/sales-order/change-status
Content-Type: application/json
Body: { "sales_order_id": "SO-001", "status": "paid" }
```

Endpoint `/change-status` bawaan platform mengharapkan body berisi primary key (atau `id`) dan `status` tujuan. Backend memvalidasi transisi, menjalankan hook `onBeforeWorkflow`/`onAfterWorkflow`, memperbarui `statusField`, dan mengembalikan `success`, `message`, `data`, serta `workflow` (status sebelum dan sesudah). Kontrak lengkap termasuk kode status error ada di [`../../api-spec/endpoint-change-status.md`](../../api-spec/endpoint-change-status.md).

`api.endpoint` boleh menunjuk endpoint lain (mis. processor kustom) selama response-nya memakai envelope `success`/`message` yang sama.

## Aturan Validasi

| Aturan | Error |
|--------|-------|
| `actionId` wajib non-empty | `"[<pageId>].workflowActions[<i>] actionId must be provided"` |
| `actionId` unik per page | `"[<pageId>].workflowActions[<i>] actionId '<value>' is duplicated"` |
| `label` wajib | `"[<pageId>].workflowActions[<i>] label must be provided"` |
| `api` wajib | `"[<pageId>].workflowActions[<i>] api must be provided"` |
| `api.endpoint` atau `api.payload` wajib diisi | `"[<pageId>].workflowActions[<i>] api.endpoint must be provided"` |
| `onError.display` hanya `modal` atau `toast` | `"[<pageId>].workflowActions[<i>] onError.display must be 'modal' or 'toast'"` |
| `confirm.summary[].field` ada di `fields[]` | warning `"[<pageId>].workflowActions[<i>] confirm.summary references field '<name>' which is not found in 'fields'"` |

Validator tidak memeriksa kesesuaian `actionId` dengan `workflow.transitions`. Aksi yang `actionId`-nya tidak muncul sebagai status tujuan mana pun tidak pernah dirender.

## Contoh Lengkap: Sales Order

Status order di form dibuat readonly agar hanya berubah lewat workflow. Aksi **Paid** dan **Cancelled** memakai dialog konfirmasi karena berdampak pada stok; **Paid** menampilkan penolakan stok sebagai modal (default).

```json
{
    "pageId": "sales-order",
    "pageTitle": "Sales Order",
    "primaryKey": "sales_order_id",
    "displayField": "order_number",
    "apiPath": "sales-order",
    "fields": [
        {"name": "order_number", "label": "Order No", "type": "text", "required": true},
        {"name": "customer_name", "label": "Customer", "type": "text"},
        {"name": "total_amount", "label": "Total", "type": "number"},
        {"name": "status", "label": "Status", "type": "text", "editorMode": "readonly", "inTable": true}
    ],
    "workflow": {
        "statusField": "status",
        "transitions": {
            "pending": ["paid", "cancelled"],
            "paid": ["shipped", "cancelled"],
            "shipped": ["completed"],
            "completed": [],
            "cancelled": []
        }
    },
    "workflowActions": [
        {
            "actionId": "paid",
            "label": "Paid",
            "icon": "check",
            "style": "success",
            "confirm": {
                "title": "Mark this order as paid?",
                "message": "Stock for every product on this order will be deducted by its line item quantity.",
                "confirmButton": "Yes, mark as paid",
                "cancelButton": "Cancel"
            },
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "paid" }
            },
            "onSuccess": {
                "notification": { "type": "success", "message": "Order marked as paid and stock has been deducted." }
            }
        },
        {
            "actionId": "shipped",
            "label": "Shipped",
            "icon": "delivery",
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "shipped" }
            },
            "onSuccess": {
                "notification": { "type": "success", "message": "Order marked as shipped." }
            }
        },
        {
            "actionId": "completed",
            "label": "Completed",
            "icon": "verify",
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "completed" }
            },
            "onSuccess": {
                "notification": { "type": "success", "message": "Order completed." }
            }
        },
        {
            "actionId": "cancelled",
            "label": "Cancelled",
            "icon": "cross",
            "style": "danger",
            "confirm": {
                "title": "Cancel this order?",
                "message": "If the order has already been paid, the deducted stock will be restored.",
                "confirmButton": "Yes, cancel order",
                "cancelButton": "Keep order"
            },
            "api": {
                "endpoint": "sales-order/change-status",
                "payload": { "sales_order_id": "$primaryKey", "status": "cancelled" }
            },
            "onSuccess": {
                "notification": { "type": "info", "message": "Order cancelled." }
            },
            "onError": { "display": "toast", "title": "Cannot cancel order" }
        }
    ]
}
```

## Interaksi dengan `fieldStates`

Workflow biasanya dikombinasikan dengan `fieldStates` untuk mengunci field tertentu setelah transisi status. Contoh: field tidak boleh diedit lagi setelah status Approved.

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

## Tombol Standar vs Workflow Actions

| Aspek | Tombol Standar | Workflow Actions |
|-------|----------------|------------------|
| Tombol yang dihasilkan | Tombol penyimpan (`{addVerb} {pageSubject}` di mode tambah, `Save Changes` di mode ubah), Cancel, Delete | Item **Change Status** di menu Actions, lalu satu tombol per status tujuan yang diizinkan |
| Endpoint | `POST /<apiPath>`, `PUT /<apiPath>/<id>`, `DELETE /<apiPath>/<id>` | `POST /<api.endpoint>` dengan body `api.payload` (default `/<apiPath>/change-status`) |
| Konfirmasi | Default: konfirmasi delete | Opsional per aksi via `confirm` |
| Tampilan error | Ringkasan error di form, modal untuk delete | `onError.display`: modal (default) atau toast menetap |
| Modifikasi data | Tombol penyimpan mengirim semua field | Workflow action hanya mengubah status |

---

← [`master-detail.md`](./master-detail.md) | [Selanjutnya: `field-states.md`](./field-states.md) →
